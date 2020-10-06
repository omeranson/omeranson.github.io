---
layout: post
title: "Endianness. But with bits."
date: 2020-10-23
---

The endianness of an architecture describes the order of bytes in which
multi-byte values are stored. But the endianness also affects the order
of bits in a single byte. I explore this notion here.

# Introduction

A while back, I got to parse some network headers on an x86. There's a funny
thing about doing this: The network data is in network order. The host works
in host order. On an x86, they don't match.

Specifically, an x86 is little-endian. Network order is big-endian.

Now, I did take this into account. I called `ntohs` and `ntohl` on all the
values. But things still didn't work.

Turns out, endianness has an effect at the sub-byte level.

# Problem Description

Let's start this with a quick introduction to endianness. Not that I assume
anyone doesn't know this, but it does give a nice place to start.

An byte is constructed of 8 bits. That means that the biggest number it can
hold is 255 (2^8 - 1). To hold numbers greater than 255, you use 2, 4, or 8
bytes together. Strictly speaking, you don't have to stick to these sizes; you
can have any size you want.

Someone might point out that the above is, strictly speaking, an octet, and
not a byte, since historically, bytes can have a different number of bits. But
these days, a byte is usually 8 bits, so I'm using that.

Now, suppose I have a number taking up 4 bytes. In what order to I place the
numbers in the bytes? Specifically, which bytes have the most and least
significant bits?

In a big-endian architecture, the first byte holds the most significant bits,
and the last byte holds the least significant bits. Personally, this seems to
me to be the intuitive way to do things. This is also the network order.

In little-endian architectures, the order is reversed. The last byte holds the
most significant bits, and the first byte holds the least significant bits. I
find this somewhat counter-intuitive. This is also the way things work on x86
architectures, and apparently, most others.

Let's illustrate this with an example. Suppose I have the 4-byte number:
`0x00010203`. The byte with the most significant bits is `0`, the second-most
significant is `1`, the next is `2`, and the byte with the least significant
bits is `3`.

The following table shows how this number is encoded in each type of
architecture:

| Big-Endian (Network Order) | Little-Endian (x86 and friends)
|----------------------------|----------------------------------
| <table><tr><td>00</td><td>01</td><td>02</td><td>03</td></tr></table>              | <table><tr><td>03</td><td>02</td><td>01</td><td>00</td></tr></table>

Functions such as `ntohl`, `ntohs`, and `bswap` exist to allow programmers to
change from one representation to another.

Now the real fun starts: The IPv4 header, for instance, starts of with two
nibbles: Version, and Internet Header Length (IHL). Each nibble is four bits,
together constructing the first byte.

This is the header, as it appears in [RFC 791, section 3.1](https://tools.ietf.org/html/rfc791#section-3.1).

```

    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |Version|  IHL  |Type of Service|          Total Length         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |         Identification        |Flags|      Fragment Offset    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Time to Live |    Protocol   |         Header Checksum       |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                       Source Address                          |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Destination Address                        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Options                    |    Padding    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

This is a trimmed definition of `struct ip`, the structure representing the
IPv4 header, as it appears in the header file `netinet/ip.h`.

```C
struct ip
  {
#if __BYTE_ORDER == __LITTLE_ENDIAN
    unsigned int ip_hl:4;               /* header length */
    unsigned int ip_v:4;                /* version */
#endif
#if __BYTE_ORDER == __BIG_ENDIAN
    unsigned int ip_v:4;                /* version */
    unsigned int ip_hl:4;               /* header length */
#endif
    uint8_t ip_tos;                     /* type of service */
    // ...                              /* Some more fields */
  };
```

Wait, what? Why are these tests here? What does it matter if the byte order is
little or big endian?

So you may be asking, who cares? Just use the header, and it works out of
the box. And the answer is, that's true. But we wanted to implement GRE. GRE
doesn't have a handy header about.

The (extended) GRE header, as is appears in [RFC 2890](https://tools.ietf.org/html/rfc2890),
has the following structure:

```
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |C| |K|S| Reserved0       | Ver |         Protocol Type         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |      Checksum (optional)      |       Reserved1 (Optional)    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                         Key (optional)                        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                 Sequence Number (Optional)                    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

If we were to define a struct representing such a header in C, it would
probably look something like this:

```C
struct gre
  {
    unsigned int gre_c:1;               /* Checksum field present bit */
    unsigned int gre__:1;               /* Reserved bit */
    unsigned int gre_k:1;               /* Key field present bit */
    unsigned int gre_s:1;               /* Sequence number field present bit */
    unsigned int gre_reserved0:9;       /* Reserved bits */
    unsigned int gre_ver:3;             /* Version */
    uint16_t gre_proto;                 /* Protocol Type */
    // ...                              /* The optional stuff */
  };
```

Which works great. On big-endian machines. On the x86 I tested this on, it
failed miserably. That's because the order of the bits came in backwards.

# Solution

The trivial solution to the above GRE header was to use the same macros as in
the IPv4 headers. I leave this as an exercise for the reader.

But the answer to the deeper question of why is: The bits are also in reverse
order. For instance, consider the number `0x11`. The following table shows the
bit representation in each endianness.


| Big-Endian (Network Order) | Little-Endian (x86 and friends)
|----------------------------|----------------------------------
| <table><tr><td>0</td><td>0</td><td>0</td><td>1</td><td>0</td><td>0</td><td>0</td><td>1</td></tr></table>              | <table><tr><td>1</td><td>0</td><td>0</td><td>0</td><td>1</td><td>0</td><td>0</td><td>0</td></tr></table>

I can hear you saying, 'well, that's nice, now prove it!'. Well all right.
Consider the following code:

```C
#include <stdint.h>
#include <stdio.h>

union byte_bits {
        struct {
                uint8_t bit0:1;
                uint8_t bit1:1;
                uint8_t bit2:1;
                uint8_t bit3:1;
                uint8_t bit4:1;
                uint8_t bit5:1;
                uint8_t bit6:1;
                uint8_t bit7:1;
        } bits;
        uint8_t byte;
};

int main(int argc, char * argv) {
        union byte_bits byte_bits;
        byte_bits.byte = 0x11;
        int idx;
        for (idx = 7; idx >= 0; idx--) {
                // The loop first prints the most significant bit, then
                // the next, and so on till the least significant bit.
                printf("%d", (byte_bits.byte >> idx) & 0x1);
        }
        printf("\t# Output from iterating over `byte`\n");

        printf("%d", byte_bits.bits.bit0);
        printf("%d", byte_bits.bits.bit1);
        printf("%d", byte_bits.bits.bit2);
        printf("%d", byte_bits.bits.bit3);
        printf("%d", byte_bits.bits.bit4);
        printf("%d", byte_bits.bits.bit5);
        printf("%d", byte_bits.bits.bit6);
        printf("%d", byte_bits.bits.bit7);
        printf("\t# Output from iterating over the struct bits\n");
        return 0;
}
```

This programme prints the bits of the number `0x11` in two ways: 

1. Iterating the bits by extracting each bit using `((value >> idx) & 1)`
   prints them in their logical order: First the most significant bit, then
   the next, and the next, until the least significant bit is reached.

2. Iterating the bits by extracting each bit using a struct. This should
   extract the bit from its physical location in the encoding of the data.
   As we will see, on big-endian machines, the bits are extracted in order.
   On little-endian machines, the bits are extracted in reverse order.

On my little-endian `x86_64` machine (an AMD 64-bit), the output is as follows:

```
00010001        # Output from iterating over `byte`
10001000        # Output from iterating over the struct bits
```

The first line's output is what we expect: The bits are in order, from most
significant to least significant. The second line shows the bits in reverse.

Following [this blog post](https://create.stephan-brumme.com/big-endian/), I
was able to run the code on a big endian, 32-bit PowerPC. The results are
below:

```
00010001        # Output from iterating over `byte`
00010001        # Output from iterating over the struct bits
```

As you can see, in this case, the bits are in the same order in both lines.

# Conclusion

As it turns out, the endianness affects not only the byte order, but also the
bit order of a value. In many cases, the CPU hides this away from you. This can
be seen in the loop that prints the first line. However, with some work, the
underlying behaviour can be seen.
