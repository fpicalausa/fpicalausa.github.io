---
layout: post
title: Reading infrared signals - Air conditioner remote control
author: fpicalausa
---

This is the third article in a series about building a remote control for a air
conditioner over the internet. Check
[Part 1](/2019/10/06/Controlling-an-air-conditioner-remotely.html) for the
overview.

To control our air conditioner, through infrared, we first need an understanding
of how the air conditioner communicates with its remote. That is, we want a
general understanding of its transport, and of its protocol.

One of the popular way of transporting signals of 0's and 1's over infrared is
called pulse distance encoding. It's also how many air conditioning units
communicate. The transport is as follows: we decide in advance of a certain
duration to encode a 1, and then a different duration (typically shorter) for
a 0. We can then send each bit in sequence. To indicate the next bit, with turn
the LED on for a fixed duration. This is illustrated in the following diagram.

![Distance Encoding]({{ site.url }}/assets/distance-encoding.svg)

The period where the LED is on is called a _mark_, while the period it is off is
called a _space_. Often times, the individual bits would be preceded by header
(essentially, a mark and space), and sometimes followed by a footer (again, a
mark and space).

## Modulation

For practical reasons, if we did turn on the LED continuously for the whole
mark, it would be hard to distinguish from background noise. This is why most IR
signals are instead sent while making the LED flick at a distinct frequency.
This frequency can be efficiently filtered by an air conditioner using a passive
band-pass filter.

38kHz is a common frequency for this. With this in mind, the above signal would
actually look as follows:

![Distance Encoding with Modulation]({{ site.url }}/assets/distance-encoding-modulated.svg)

Fortunately, most infrared receivers actually decode this 38kHz signal into
larger marks and spaces, so we rarely have to deal with the raw signal.

## Receiving IR signals

Here are the components, and circuit I used to test out if I could receive IR
signals from the air conditioner remote control.

- Raspberry Pi 3b+
- A breadboard, a Raspberry Pi breadboard connector, a 10 wires
- (IR receiver) VS1838B infrared receiver module

![Infrared reader]({{ site.url }}/assets/infrared-read.svg)

## Configuring the Raspberry Pi

Raspbian comes with support for the popular LIRC project, that supports many
infrared input and output.

First, install LIRC:

```sh
sudo apt-get install lirc
```

Enable the IR interface in your `/boot/config.txt`:

```txt
dtoverlay=gpio-ir,gpio_pin=17
```

The gpio_pin in the above corresponds to the pin connected to the OUT node of
the IR received.

After rebooting your raspberry pi, you should have a new `/dev/lirc0` device.
You can check if you can receive infrared signals by using the `mode2` tool,
which will helpfully write the different marks (pulses) and spaces duration:

```sh
mode2 --device /dev/lirc0
```

After pressing any button on the remove, you should get an output similar to the
following:

```
Using driver default on device /dev/lirc0
Trying device: /dev/lirc0
Using device: /dev/lirc0
space 16777215
pulse 3360
space 1571
pulse 490
space 381
pulse 463
```

## References

Protocol overview

1. [NXP's description of IR protocols](https://www.nxp.com/docs/en/application-note/AN3053.pdf).
   The Distance Encoding is further explained in this document.
2. [This instructable](https://www.instructables.com/id/Reverse-engineering-of-an-Air-Conditioning-control/)
   describes the process of reading infrared signals.
3. LIRC [mode2 man page](http://www.lirc.org/html/mode2.html)
