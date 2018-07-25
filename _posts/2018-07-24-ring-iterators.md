---
layout: post
title:  "Radical Ring Iterators"
date:   2018-07-24  00:00:00
---


## Intro
I admit I have a bit of an obsession with Python which may seem a bit strange
for someone who does most of their work with embedded systems. However, every
now and then I'll have a stroke of clarity when I am able to apply a high level
concept to solve a low level problem I'm facing. This is a fantastic example.

Before you read any further, I would really like to give a shoutout to Wade
Fagen and the CS 225 crew to opening my eyes up to programming, but especially
the invaluable practice of using data structures to transform the way we view
and solve difficult problems. Specifically there was a MP (Project) in the
class where we implemented BFS and DFS iterators to color images. While
I struggled for at least 30 hours to understand how C++ and range based
iterators work, it was one of the first times where I think I saw beyond the
basics of iterative/sequential programming and into the beautiful art of
transforming problems into trivial solutions via paradigms and composition.
And I guess that was a good thing because the CS 225 is the Data Structures
Course! :p

[CS 225 MP4: Flood Fill][mp4-link]

![My Floodfill!](https://raw.githubusercontent.com/epellis/epellis.github.io/master/floodfill.gif)

## The Problem
Recently, I have been working on a serial console for a remote job and needed
to implement a doubly ended queue that could buffer potentially hundreds of
serial messages while still being statically allocated, with O(1) removal and
insert operations. Worse still, the buffer may also need to quickly reset its
head or tail and advance or reset the header to a multiple of the message size.

## Implementation #1
My first thought was to work with a statically allocated array with head and
tail pointers, as was taught in my ECE 220 Class.

```cpp
typedef struct {
    uint16_t head;
    uint16_t tail;
    uint16_t buf[BUF_LEN];
} ringbuf_t;
```

While this implementation was certainly workable, it had several problems:
- No easy way to check the size of the ring buffer
- Enqueue/Dequeue was a verbose, two step process:
    1. Copy variable to array
    2. Increment index variables
- All math logic was done on the "client side"
- Index rewrapping (e.g. idx = 5, 6, 7, 0, 1, 2...) used modulo (%) which is
  very slow
- C does not have a generic container structure so only uint16's could be
  stored
- No way to compare indices reliably

## Implementation #2
To fix these issues, I ended up making a RingBuf file with associated functions
for most operations as well as static helper functions and internal data types.
More or less, it looked like:

```java
uint16_t ringbuf_remaining(ringbuf_t* buf)
{
    return abs(buf->head - buf->tail);
}

uint16_t ringbuf_pop(ringbuf_t* buf)
{
    uint16_t data;

    data = buf->buffer[buf->tail];
    buf->tail = (buf->tail+1) % buf->size;

    return ptr;
}

uint16_t ringbuf_peek(ringbuf_t* buf)
{
    return buf->buffer[buf->tail];
}

void ringbuf_push(ringbuf_t* buf, uint16_t data)
{
    buf->header[buf->head] = data
}

void ringbuf_reset(ringbuf_t* buf)
{
    buf->head = 0;
    buf->tail = 0;
}
```

This approach made some signifigant improvements by standardizing the ring
buffer functions (and thus their errors) but still had one glaring issue you
can hopefully see - most of the functions have two lines which are duplicates
of each other. The first one applies the operation to the "head" and the other
applies an almost identical operation to the "tail". Now since this is an array
with two indices, this makes a good bit of sense, as both indices behave in
a very similar way. Unfourtonately this made writing code not only error prone
but also very tedious and verbose.

Also for those of you who are wondering why the modulo is still present, as
they say, "Premature Optimization is the Root of all Evil!!!" Later
implementations did some simple bounds checking which I left out for brevity.

## Python to the rescue!
One of Python's many philosophies is to implement as many complex data
structures out of simpler, atomic data structures given to you by the language.
While C does not provide many of these, it does make more sense to compose our
doubly ended ring buffer out of three components:

1. C-Style array that holds data
2. Head index
3. Tail index

Not only does this allow for the underlying type for component #2 and #3 to be
reused, but you can even swap out component #1 (the array) for whatever type
you want. This was a great benefit because I was also working to adapt the
doubly ended ring buffer to other data types such as structs. What could have
been tedious hours of pointer arithmetic and hundreds of lines of code turned
into a simple redeclaration at the top of my file.

But this alone dosen't account for my title, "Radical Ring Iterators". By
composing my data structure out of two seperate components, I only shifted my
problems from this data structure to the one implementing the index. However,
this is where the wisdom of Python really shines. Any pythonista knows that
almost all iteration in python does not use indices but rather what they call
iterators, constructed from either generators or lists. Here is an example of
two ways to iterate through a list:

```python
# Not Pythonic :(
for i in range(len(list)):
    print(i)

# Pythonic :)
for item in list:
    print(item) 
``` 

As the better Ned (Ned Batchelder) says, implementation #1 is a "rube goldberg"
approach to programming. Even though we are only interested in the items in the
list, we have this strange idea of first transforming our iteration to
integers, and then back to items. Python realizes this and implements the
iterator as a powerful default aspect of the language.

Sadly, C has no standard implementation of the iterator, but that dosen't mean
we can't make one. This was probably the most mundane part because it ended
being very simple, just copying and pasting functions with small changes.

## Implementation #3
Lets walk through the final result. First up are the data structures as they
will be referenced later in the review:

```java
typedef struct riter_t {
    uint16_t idx;
    uint16_t size;
    int16_t stepsize;
    uint16_t cycles;
} riter_t;

typedef struct dequeue_t {
    riter_t head;
    riter_t tail;
    uint16_t* buf;
} dequeue_t;
```

Surely enough, you can see that now the doubly ended ring buffer also contains
two ring iterators (riters) which represent the head and the tail indices.
A closer look at the riter struct shows us some interesting characteristics,
which I will explain: 
- `idx`: The index. Pretty self explanatory
- `size`: The size of the buffer. Could be anywhere from 8 to as much as 8192
- `stepsize`: Ah now here is where things get interesting. As many of you know,
  Python list slicing gives you three options: start, end and step. By making
step=2, for instance you can loop over every even element in the list. This is
very helpful for us, since sometimes I want my array to increment by the
message size instead of 1. By making it a integer, you could even make it
increment backwards, if for example, you wanted to implement some kind of LIFO
(Stack) type.
- `cycles`: Since ring iterators never stop, they need a way to be compared to
  each other. By counting the amount of "cycles" an iterator has traveled, you
can track how many overflows and add that to the total distance. Since some of
my algorithms need to see how far two riters are away this attribute is
critical.

You can also see the dequeue here. While I was lazy and left it as a uint16
pointer, there is nothing stopping me from making a different struct with
something else (like a `double`) as the base data type. Next, lets take a look
at the function prototypes:

```java
/* ======== RING ITERATOR (RITER) ======== */
/*
 * Manipulates an iterator that can be used as the index of
 * one (or more) buffers. Common uses include implementing
 * a stack, queue, ring buffer or other linear data structure.
 *
 * Riters are often paired with the dequeue (doubly ended queue)
 * as uint16_t buffers but it is completely possible to represent any
 * ADT by setting dequeue.buf = NULL and implementing the access
 * operators yourself.
 */
uint16_t RiterIdx(riter_t* it);
uint16_t RiterOffset(riter_t* it, int16_t offset);
uint16_t RiterAdvance(riter_t* it);
uint16_t RiterReset(riter_t* it);
uint16_t RiterSet(riter_t* it, uint16_t new_idx);
int16_t RiterDist(riter_t* it_a, riter_t* it_b);
void RiterSetStep(riter_t* it, uint16_t new_step);
```

How fantastic! I don't see a single case of a duplicated function. By keeping
our data structures fundemental we allow a greater level of flexibility, as
I could use this to implement a singly ended structure such as a queue or
stack, as well as use the riter alone as some kind of cyclical timer or
counter. Really the possibilities are endless.

## Conclusion
Hopefully this inspires you to try to integrate features of different fields
in hopes of simplifying issues and solving some of your problems. As this
project draws closer to release I hope to keep delivering nuggets of advice and
learn ways you have also used this multi-paradigm approach to solving problems.
I really can't thank the few people who took tens of hours to laboriously show
me why small articles and ideas such as this one are so neat and would love to
have that kind of impact on someone else someday.

Until Next Time,

    - epelesis

[mp4-link]:https://courses.engr.illinois.edu/cs225/fa2017/mps/4/
