---
layout: post
title: "Register manipulation: quick and dirty PoC"
description: "A very quick and dirty prototype for manipulating registers for an LED matrix display."
modified: 2015-03-07
tags: [code, embedded, LED matrix clock]
image:
  feature: digit_design.jpg
---

I've recently started a small project building a digital clock which uses a 16x5 matrix of LEDs for display. The LEDs are switched one row at a time, with each of the individual 16 LEDs per row being addressed via shift registers. I wanted to write a quick program as a proof of concept, which would emulate the LED matrix with 1s and 0s in the terminal. Essentially it's just an easy way to see if my code is going to work without having to have the hardware at hand.
<!--more-->

### A primer on shift registers

This really isn't in the scope of this post, but I figured it was worth touching on the basic principle. This project will utilize a latched SIPO (serial input, parallel output) shift register.[^1] This buffers serial data until a latch is activated, at which point the buffered bits are copied to the output registers. In essence it allows a microcontroller to use a small number of wires (serial input) to set the state of many wires (parallel output).

[^1]: [74HC595 Shift Register](http://www.nxp.com/documents/data_sheet/74HC_HCT595.pdf) PDF Warning!

### Representing the digits

As the display is driven one row at a time, with each digit occupying 3 LEDs on each of the 5 rows, we need to first create a map representing each number we want to display. It seemed most logical to me to map each number as an array of which LEDs should be active per row. I originally did this in binary, as it's the most obvious representation, however C doesn't _actually_ support binary integer constants.[^2] I mapped these out by hand in binary, and then converted them to hex. Here's what I came up with:

[^2]: [Rationale for International Standard - Programming Languages - C: §6.4.4.1](http://www.open-std.org/jtc1/sc22/wg14/www/C99RationaleV5.10.pdf) PDF Warning!

{% highlight c %}
const uint8_t num[10][5] = {
  {0x07,0x05,0x05,0x05,0x07}, // 0
  {0x01,0x01,0x01,0x01,0x01}, // 1
  {0x07,0x01,0x07,0x04,0x07}, // 2
  {0x07,0x01,0x07,0x01,0x07}, // 3
  {0x05,0x05,0x07,0x01,0x01}, // 4
  {0x07,0x04,0x07,0x01,0x07}, // 5
  {0x04,0x04,0x07,0x05,0x07}, // 6
  {0x07,0x01,0x01,0x01,0x01}, // 7
  {0x07,0x05,0x07,0x05,0x07}, // 8
  {0x07,0x05,0x07,0x01,0x01} // 9
};
{% endhighlight %}

With this we have an easy way of finding the binary representation for each row of a given digit. For example the first row of the number "3" would be `num[3][0]` which will evaluate to `0x07` or `0b00000111`.

### Deciding on a data structure

Now that we have a representation of each number, we need to decide how to represent a 16-bit register for each row. There's a few caveats we must consider here. Firstly, we would ideally like to be able to address each of the 16-bits in the software register individually. Secondly, in the actual project, there will be two 8-bit shift registers connected serially. As such, it makes the most sense to have a method of breaking our 16-bit register in software into two 8-bit pieces to send into the shift registers (it's actually not sctrictly necessary to do this, but I feel it's good practice).

With these points in mind, it seems the most sensible thing to use here is a union containing a bit-field structure. This way we can address the data in any of three ways: per bit, per byte, or per word.

{% highlight c %}
struct BIT_INFO {
  uint16_t s4 :1; // bit 0
  uint16_t d4 :3; // bit 1-3
  uint16_t s3 :1; // bit 4
  uint16_t d3 :3; // bit 5-7
  uint16_t s2 :1; // bit 8
  uint16_t d2 :3; // bit 9-11
  uint16_t s1 :1; // bit 12
  uint16_t d1 :3; // bit 13-15
};

union HardwareRegister {
  unsigned char   bytes[2];
  uint16_t        word;
  struct BIT_INFO bits;
};
{% endhighlight %}

Note that in the bit-field I've included a single bit between each 3-bit "digit" placeholder. This will make the visual representation look clearer in the finished project by leaving a column of LEDs dark between each digit.

### Assembling the register

Next we need to define a method for "assembling" our register in software. Since we're going to be representing time, our input data will presumably be 2 integers: one integer for hours, and a second for minutes. As we only have a map for numbers 0-9, we need to break down the time integers into single digits, which is easy enough using the modulus function:

{% highlight c %}
uint8_t h1 = (hour / 10) % 10;
uint8_t h2 = hour % 10;
uint8_t m1 = (min / 10) % 10;
uint8_t m2 = min % 10;
{% endhighlight %}

With the time now represented with 4 individual single digit integers we can easily assemble the register. To save on memory space, we'll only declare one register and then loop through each row setting the register and writing the output 5 times. This is what it looks like:

{% highlight c %}
union HardwareRegister reg;

int i;
for(i=0; i < 5; i++)
{
  reg.word = 0;
  reg.bits.d1 = num[h1][i];
  reg.bits.d2 = num[h2][i];
  reg.bits.d3 = num[m1][i];
  reg.bits.d4 = num[m2][i];

  writeRow(i, &reg);
}
{% endhighlight %}

### Displaying the register

One of the stated goals of this post was to be able to display the contents of the register for verification. To do this we need to be able to step through our register and print each bit individually. Rather than reinvent the wheel I came across a very nice snippet online for doing exactly this, and best of all it's completely type agnostic, so it works just fine with our union.[^3]

[^3]: <http://stackoverflow.com/a/3974138>

{% highlight c %}
void printBits(size_t const size, void const * const ptr)
{
  unsigned char *b = (unsigned char*) ptr;
  unsigned char byte;
  int i, j;

  for (i=size-1;i>=0;i--)
  {
    for (j=7;j>=0;j--)
    {
      byte = b[i] & (1<<j);
      byte >>= j;
      printf("%u", byte);
    }
  }
  puts("");
}
{% endhighlight %}

with all this put together, we can verify our representation of any given pair of two digit numbers (i.e. time). As an example, I've passed "12" & "34", as hour and minute respectively, to our routine, here's what the output looks like:

~~~
0010111011101010  
0010001000101010  
0010111011101110  
0010100000100010  
0010111011100010  
~~~

Now we have verification that our 16-bit registers are appropriately representing the serial data we need to display our digits on a matrix.

### Wrapping up

I find writing quick little programs like this as a proof of concept for an algorithm to be much easier than going straight to a microcontroller - not to mention I can do it without having the hardware in front of me. Now there's a working prototype which can easily be ported over to a microcontroller for the final project. We'll look at that in an upcoming post.

Full code for this little demo / PoC is available on my github, here: [LED Matrix Demo](https://github.com/dstreby/LED_matrix/blob/master/src/demo.c)

Thanks for reading!

### Addendum:

It should be noted that the code and methods used above assume the program will be running on a system using little endian. This is fine for x86 systems, as well as AVR (except AVR32) and PIC MCUs, however modifications would need to be made for Freescale / 68xx, etc.

***
