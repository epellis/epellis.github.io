---
layout: post
title:  "Project Spectra: Part I"
date:   2017-01-02  00:00:00
---

## Introduction

The past few years of browsing websites such as Hackaday, Instructables and
r/raspberry_pi have given me a frankly unhealthy fascination with LED Matrices.
While one might disparage them for their paltry resolution or their high power
draw, I have always loved their ability to be seen both as art and electronics.

After finishing my first semester at Illinois, I have felt that my technical skills
(and patience) have finally progressed to a level in which I would feel confident
making one of my own. However, to avoid falling down the dangerous and deadly trap
of procrastination, I am doing a few things differently than most of my projects.

Firstly, I will not be the only one using one of these matrices. One of my friends
birthdays is approaching and I think it would be great to both inspire and amaze
her with a personally fabbed project. Ideally, it would be reasonably hackable
so that she could in the future could implement her own functionality.

Additionally, I would like to not only highlight skills I already know, but also
build new ones. Things such as CAD design, PCB design, Vector design and ESP8266
development are all fairly new fields to me and I am a very strong believer that
the best way to learn is to make.

Thus, my requirements are as follows:
- Totally complete and ready to ship by early March
- Contains at least one self-designed PCB
- Uses internet connectivity to augment features
- Features at least one instance of digital signal processing
- Housed in a custom designed, 3D printed shell
- Exhaustively blogged and well documented

## The First Function: Climate Display

Since I had already dabbled in creating a basic gradient in my AP Computer
Science class last year, I figured it would be as a jumping off point as any and
began to develop the first of four display functions. If all of this sounds vague
and unfinished, don't worry, that's because it is. Expect another post in the near
future with some more concrete goals.

The Climate Display should feature
- A gradient that is directly influenced by the local temperature, humidity and wind speed
- An animation of the moon and it's phases, directly influenced by the current phase of our moon.

## Processing

A good design step to invoke is to simulate your project before you build it.
Since my last semester has been pretty "bare metal" heavy, I wanted to ensure
that my ability to write complex algorithms and firmware was at least somewhat
intact. That meant dusting off my favorite graphics programming language,
[Processing][processing-site]. Since it uses Java, which I have a good deal of
experience in, and is very popular in the tutorial and DIY market, Processing is
an ideal platform to write some basic rendering and algorithm tests.

### Step 1: Render our Matrix

The first step was to generate something that looked vaguely like the 8x8 Matrix
my project will be using and render a random colorset to it. Here is a snippet
of code that individually assigns each pixel a random color.

You can find all of my rendering code at [Github][github-repo-link]

'''java
/* Iterate through each subpixel (e.g. red, green, blue) */
for (int i = 0; i <  display.length; i++) {
  for (int j = 0; j <  display[0].length; j++) {
    for (int k = 0; k < 3; k++) {
      /* Apply color biasing */
      if (k == 0) {
        display[i][j][k] = targetRed + (int)(Math.random() * 18 - 9);
      } else if (k == 1) {
        display[i][j][k] = targetGreen + (int)(Math.random() * 18 - 9);
      } else if (k == 2) {
        display[i][j][k] = targetBlue + (int)(Math.random() * 18 - 9);;
      }

      /* Make sure value is within [0, 255] */
      if (display[i][j][k] < 0) {
        display[i][j][k] = 0;
      } else if (display[i][j][k] > 255) {
        display[i][j][k] = 255;
      }
    }
  }
}
**/
'''

As you can see, this code takes a bit of processing power to push since it has to
generate a random number for EVERY SUBPIXEL. Some quick math means that for every
frame, that's 192 numbers. Also, this code does not scale well to higher resolutions something that I found later to be extremely helpful for getting a bigger picture of my algorithm.

Spectra Test V2's sole goal was to transition the dotted screen display into discreet squares and also make the algorithm scalable for any resolution. Here is a snippet of the new code used to render a matrix to screen.

'''java
void draw() {
  /* Iterate through each pixel and render it to screen */
  for (int i = 0; i < LEDCount; i++) {
    for (int j = 0; j < LEDCount; j++) {

      /*
        Code to render a rectangle
        Corner 1: HEIGHT * (i / LEDCount), WIDTH * (j / LEDCount)
        Corner 2: HEIGHT * ((i + 1) / LEDCount), WIDTH * ((j + 1) / LEDCount)
      */
      int x1 = (int)(height * i * (1.0 / LEDCount));
      int y1 = (int)(width * j * (1.0 / LEDCount));
      int x2 = (int)(height * (i + 1) * (1.0 / LEDCount));
      int y2 = (int)(width * (j + 1) * (1.0 / LEDCount));      
      /* Set fill to color at pixel located at (i, j) */
      fill(display[i][j][0], display[i][j][1], display[i][j][2]);
      rect(x1 , y1, x2, y2);  // Draw the rectangle
    }
  }
}
**/
'''

As you can see, this code is not only much faster due to it's lack of floating
point operations, but also totally scalable. LEDCount specifies the resolution of
the screen, so in this case it was set to 8.

### Step 2: Generate a Basic Gradient

Spectra Test V3 was when all of the pieces began to finally fall into place.
I spent about a day messing around with different mathematical equations to
successfully render a matrix (shoutout to Wolfram Alpha), but in the end, to
keep our code simple and quick, I stuck to the boring diagonal linear gradient.
To render this, two one-colored arrays were generated and then each pixel's symmetrical complement was averaged. While there are no doubt easier ways to make
this a possibility, I rather liked the modular function of the code as it made it
easy to debug. Here is the part of my code that generated and averaged the arrays.

'''java
/* Assign initial values to display by iterating through each subpixel */
for (int i = 0; i < LEDCount; i++) {
  for (int j = 0; j < LEDCount; j++) {
    colorB[i][j][0] = (int)(redB * (intensity / LEDCount) * (i + j));
    colorB[i][j][1] = (int)(greenB * (intensity / LEDCount) * (i + j));
    colorB[i][j][2] = (int)(blueB * (intensity / LEDCount) * (i + j));
  }
}

/* Crossfade the pixels by averaging them */
for (int i = 0; i < LEDCount; i++) {
  for (int j = 0; j < LEDCount; j++) {
    display[i][j][0] = (colorA[i][j][0] + colorB[LEDCount - i - 1][LEDCount - j - 1][0]) / 2;
    display[i][j][1] = (colorA[i][j][1] + colorB[LEDCount - i - 1][LEDCount - j - 1][1]) / 2;
    display[i][j][2] = (colorA[i][j][2] + colorB[LEDCount - i - 1][LEDCount - j - 1][2]) / 2;
  }
}
**/
'''

![]({{site.baseurl}}/images/SpectraRender/gradient3.png)

![]({{site.baseurl}}/images/SpectraRender/gradient5.png)

> Here you can see my first try at an algorithm, note the black corners.

![]({{site.baseurl}}/images/SpectraRender/gradient10.png)

![]({{site.baseurl}}/images/SpectraRender/gradient11.png)

![]({{site.baseurl}}/images/SpectraRender/gradient12.png)

> The new linear algorithm with some nice colors from UIGradients.

### Step 3: Bring on the Dope Tech

Spectra Test V4 took the greatness that V3 had and applied some easier ways to
interpret a RGB Value and also added back in some of the random noise that V1 had
characterized.

'''java
/* Set fill to color at pixel located at (i, j) */
fill(display[i][j][0] + (random.nextInt(boundaries / 2) - boundaries), display[i][j][1] + (random.nextInt(boundaries / 2) - boundaries), display[i][j][2] + (random.nextInt(boundaries / 2) - boundaries));
stroke(display[i][j][0], display[i][j][1], display[i][j][2]);
rect(x1 , y1, x2, y2);  // Draw the rectangle
**/
'''

Spectra Test V5 is a new area for me, as only 50% of the activity cycle is spent rendering a gradient. The rest is spent rendering a crescent shaped moon, which
is supposed to resemble the actual moon phase of that night. To do that, I once
again called upon Wolfram Alpha and started off generating some code to generate
a circle. Here is some code that is essentially calculating the radius and then
deciding if it is worth it to fill in the associated pixel.

![]({{site.baseurl}}/images/SpectraRender/Moon1.png)

> A very simple rendering of a circle.

## Conclusion

This week, I will be working on perfecting the moon generation code and beginning
to develop some basic physical designs. Specifically, I am hoping to begin to learn
how to use Altium to build PCB's, and continue to use SolidWorks to make a CAD
mockup of my final shell so that I can get a physical idea of what dimensions I'll
be working with. Stay Tuned!

[github-repo-link]: https://github.com/epellis/Spectra/tree/master/Processing
[processing-site]: https://processing.org/
