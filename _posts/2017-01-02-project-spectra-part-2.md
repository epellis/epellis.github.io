---
layout: post
title:  "Project Spectra: Part II"
date:   2017-01-30  00:00:00
---

## One Month In

After looking at the date that the last post was published, I have realized that
I should really be getting into the habit of posting more. Needless to say, school
and organizations have kept me fairly busy and only now have I found a second to
give a quick status report about all of my undertakings.

## Making a Printed Circuit Board

Since the main goal of this project was to become comfortable designing systems
and integrating them into one centralized board, I began work creating my PCB
rather early. As this is my first project creating a PCB, I was unsure which
software I should use, and after consulting a few friends decided upon KiCad which
offers a completely free experience while still giving me the tools I would need
to properly make a PCB.

I started the process by listing the components in a "Block Diagram" manner,
allowing for me to understand which items would require interfacing with the
Microcontroller. This step was difficult because I still didn't have a solid idea
of what I would be using for the final project and thus the original list was pretty
vague. In the end I came up with something like this.

Basic Parts:

  - ESP 8266 Microcontroller
  - WS2812B LED Matrix
  - 3.3V --> 5V Logic Level Shifter
  - 3.3V Voltage Regulator
  - MSEQ7 Digital Waveform Processor
  - Push Buttons (2)
  - Rotary Potentiometer
  - Power Switch

After coming up with this list I began to plan out my design in KiCad. Although
the first iteration was a jumbled mess of wires I was able to get some much needed
advice and criticism from one of my friends who taught me the basics of good
schematic design and how to make your circuit readable. Here's an in-progress
shot of what I've got so far:

![]({{site.baseurl}}/images/SpectraTesting/KiCad.jpg)
![]({{site.baseurl}}/images/SpectraTesting/fourLED.jpg)

## Ordering the Parts (Alternative title: Blowing Money Fast)

Once I had a basic idea of what the final product would consist of I began to
order parts to both prototype with and hopefully recycle as part of the completed
product. The problem was that although the final iteration of Spectra should hopefully
be very cheap (<$40), I had to buy a couple of tools to correctly assemble the
circuit and ensure that everything would actually come together cohesively. In the
end I ended up getting the following on Amazon.

Shopping Cart:

  - 8x8 WS2812B Matrix (Kuman not Adafruit)
  - 10 x 1000uF Capacitors
  - Quad Level Shifter
  - Wire Stripper/Cutter
  - 22 Gauge Wire Reel (Multiple Colors)
  - Digital Multimeter
  - Banana Plug to Alligator Wires

Combined with my already existing and rapidly expanding electronics kit I was now
set to prototype. I will include prices and vendors in the final writeup.

## Entering the Hell that is Known as Debugging

Now came the moment of truth. The first thing I wanted to test was that the
ESP8266 would be able to output the signal required to drive the WS2812B's, as
this functionality was really the core of the whole project. To start I hooked
the output pin on the ESP to an oscilloscope to ensure that the timings were
within the very picky ranges that the WS2812B likes to see in order to function
correctly. Although the testing part worked brilliantly, I found it disappointing
that the [$3000 Keysight][Keysight-Site] we had on hand failed to recognize the USB
card and thus did not provide any screenshots of the signal acquisition process.

To generate the signal, I used the popular and widely ported [FastLED Libarary][FastLED-Site]
to send the right signals to the Ws2812B. Here is a snippet of the starter code
I used to test the matrix. To find all of my code, head to [Github][Github-repo-link].

```C++
void setup() {
  FastLED.addLeds<CHIPSET, DATA_PIN, COLOR_MODE>(leds, NUM_LEDS);
  for (int i = 0; i < NUM_LEDS; i++) {
    leds[i] = CRGB::Black;
  }
  FastLED.show();
  delay(1000);
}

void loop() {
  for (int i = 0; i < NUM_LEDS; i++) {
    leds[i] = CRGB::Magenta;
    delay(DELAY);
    FastLED.show();
    delay(DELAY);
  }
  for (int i = 0; i < NUM_LEDS; i++) {
    leds[i] = CRGB::Teal;
    delay(DELAY);
    FastLED.show();
    delay(DELAY);
  }
}

```

## Problems Continue to Arise as Testing Continues

Right away, I discovered that for an unknown reason, the ESP was only outputting
the signal for the first 4 LEDs. This shortcoming leads me into discussing one of
the largest difficulties this project possesses. Although the ESP8266 seemingly
functions on the same code as the Arduino Uno, they are not similar microcontrollers.
The Arduino Uno features an Atmel 328p, which for all purposes will simply execute
each line of code sequentially and loop back as required. This means that when libraries
are designed for the Uno that require very specific timings, the microprocessor can
more or less meet every deadline immediately. The ESP 8266 uses a totally different
microcontroller (Tensilica L106) with a different control scheme, notably the use
of a Real Time Operating System or RTOS. This essentially gives it the power to
schedule and prioritize tasks allowing for higher level functions to execute faster
and more reliably. However, in the process, since a program can randomly be interrupted
by a higher task, the ESP can sometimes not provide the same level of timing accuracy
that the Arduino Uno using the Atmega 368 can. Thus to continue testing my LED
display, I decided to temporarily use an Uno so that my output would already be at
5V and my logic slightly more reliable as the Arduino is a better understood and supported
platform when the WS2812B IC is concerned.

![]({{site.baseurl}}/images/SpectraTesting/fourLED.jpg)

Once I switched microcontrollers, my number of problems immediately decreased.
Thus, I continued testing the FastLED library to ensure that my panel had no defects
and that the codebase would be sufficient for my plans. I also wanted to give the
display a real-world power consumption by hooking it up to a power supply which
monitored both output voltage and current. Thanks to ohm's law, we know that
power = voltage * current, and thus by measuring both I could get a ballpark estimate
on the amount of wattage I would need to supply to my system. The WS2812 is a
particularly special case because each LED is powered independently of each other.
This means that making the entire display red takes approximately half the power
it takes to make the entire display purple since the purple light requires the
red and blue LED to be lit. Thus to test the maximum consumption of the device
I set each IC to output pure white. This resulted in a power draw of 2.22A at
4.99V. Some simple multiplication tells us that this panel therefore consumes
a maximum of around 11 watts of power. To play it safe and put a significant
amount of headroom, that means opting for a 5V, 3A power supply putting out a
maximum of 15 watts, plenty to power both the display and any electronics on the
board.

![]({{site.baseurl}}/images/SpectraTesting/powerSupply.jpg)

Lastly, to confirm that this display would be sufficient I wanted to test the
color accuracy on it. To do this, I would tell the WS2812B to display a hex color
and then replicate the same color on my computer. After sending a few colors to
the display I confirmed that although the display is indeed accurate, making the
color too bright will overpower the human eye, rendering the color white. This is
probably a good time to mention that the entire panel at full scale is BRIGHT.
Any concerns that I used to have about the ability of this device to make a
noticeable effect in room-level brightness was vanquished and I am now curious as
to how much the diffuser I will be putting on the case will effect the final result.

![]({{site.baseurl}}/images/SpectraTesting/blueScreen.jpg)

## Hangups and Skips

Once the panel was verified to not be DOA, I began to run some example code on
it to see how well it would respond to animations. The answer is not well.
While this might be an individual hardware issue, I found numerous examples of
the panel freezing in the middle of an animation, or worse, rapidly blinking a
second color. Hopefully this was a connection based issue, although I will
certainly be confirming that this issue is gone before going ahead and printing
my board. Next, I will be focusing on integrating the ESP and the WS2182 and if
all goes well, beginning to test the input controls to cycle through animations.

[github-repo-link]: https://github.com/epellis/Spectra/tree/master/ESP8266
[Keysite-site]: http://www.keysight.com/en/pdx-x202183-pn-MSOX3024T/mixed-signal-oscilloscope-200-mhz-4-analog-plus-16-digital-channels?cc=MY&lc=eng
[FastLED-Site]: http://fastled.io/
