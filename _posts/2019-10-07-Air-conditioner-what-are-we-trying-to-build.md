---
layout: post
title: What are we trying to build? - Air conditioner remote control
author: fpicalausa
---

This is the second article in a series about building a remote control for a air
conditioner over the internet. Check
[Part 1](/2019/10/06/Controlling-an-air-conditioner-remotely.html) for the
overview.

Our system should be able to change the air conditioner settings from a web
interface. I have a Raspberry Pi 3b+, so I will be using it for this project.
Now, changing the settings is useful, but without a good idea of the current
temperature in the room, it is difficult to know remotely if turning the air
conditioner on makes sense. So we'll also want to build that.

My air conditioner is a Fujitsu AS-E4007T. It doesn't come with a web interface
(otherwise I wouldn't be writing this), but it does come with an infrared
interrace (a plain looking remote control). Fun fact: most phone cameras will
pick up infrared. Point your phone camera at the remote control LED, press a
button on the remote, and the camera should show the LED blinking.

Given these, here is my mental model of what I wanted:
![Overview diagram]({{ site.url }}/assets/remote-control-overview.svg)

In addition to the transmitter, we'll also need to be able to read (and decode!)
infrared signal from the original remote control, so that we know what signals
are expected.

## Bill of material

For this project, here are the components I used:

-   A Fujitsu AS-E4007T air conditioner
-   Raspberry Pi 3b+
-   A breadboard, a Raspberry Pi breadboard connector, a 10 wires
-   (IR transmitter) Vishay TSHF5410 infrared LED
-   (IR transmitter) Fairchild 2N4401 transistor
-   (IR transmitter) 1.5 kOhm resistor
-   (IR transmitter) 39 Ohm resistor (x2)
-   (Temperature probe) DHT11 module, mounted with a filtering capacitor and
    pull-up resistor
-   (IR receiver) VS1838B infrared receiver module

The temperature probe and infrared receiver where bundled with an electronics
starter kit I bought earlier, so not much went into deciding what to use there.
The transmitter was another story.
