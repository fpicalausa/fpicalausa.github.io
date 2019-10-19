---
layout: post
title: Controlling an Infrared LED - Air conditioner remote control
author: fpicalausa
---

This is the fourth article in a series about building a remote control for a air
conditioner over the internet. Check
[Part 1](/2019/10/06/Controlling-an-air-conditioner-remotely.html) for the
overview.

In the previous article, we saw how Distance Encoding is used to transport
information over infrared to an air conditioner. This time, we'll see how to
control an infrared LED so that it can generate such signal.

In my case, I bought a Vishay TSHF5410 LED, mainly because it was available at
my local electronics store, but also the datasheet recommends it for:

> Infrared high speed remote control and free air data transmission systems with
> high modulation frequencies or high data transmission rate requirements

Our requirements aren't that high, but fits the bill all the same.

To generate a signal at 38kHz, using Distance Encoding, we will need to write
the corresponding signal to one of the GPIO pin of the Raspberry Pi. This in
turn needs to pulse the LED at the right frequency, while keeping it off the
right amount of time to encode 0's and 1's.

## Powering the LED

Powering a LED is usually as simple as making sure that you can provide the
right current to it. The more current provided, the higher the radiant power you get
(that is the LED gets "brighter"). Going over a certain current, however, will
damage the LED. The specific current depends on the pulse length and how fast
the pulse is repeated.

Since we are sending IR signals at 38kHz, we repeat the pulse every
$T = 32 \mathrm{\mu s}$. The plan is to turn the LED on for half the time or $t_p = 16\mathrm{\mu s}$. Looking at the datasheet, for $t_p/T = 0.5$,
and a pulse of $16\mathrm{\mu s}$, a current less than 200mA won't damage the LED.

The GPIO pins of the rasberry pi can only provide 16mA maximum without being damaged. 
The 5V pins are linked directly to the
power source, allowing to draw more current. This means that we need to power
the LED through a 5V pin, while controlling the signal from one of the GPIO.

## Transistor controlled signal

A transistor can be thought of as a current-operated switch. Transistors have three
connections: the base, the emitter and the collector.

- When there is no current at the base, no current flows between the emitter
  and the collector
- When there is a (small) current at the base, current can flow from the
  collector to the emitter.

If we connect one of the GPIO of the Raspberry Pi to the base, and set the LED
between the collector and the 5V provided by the Raspberry Pi, we will
effectively be able to control the LED.

How much current is needed at the base is indicated by the $H_{FE}$ coefficient. 
This is the ratio of current going from collector to the emitter over the current 
going through the base.

## Circuit

The diagram is as follows:

![IR LED circuit]({{ site.url }}/assets/npn-ir-led.svg)

On a breadboard:

![IR LED circuit]({{ site.url }}/assets/irled-board.svg)

I added two resistors to limit the current going through the LED, and the current drawn from the Raspberry Pi.

The current flowing through the LED should be 200mA or less. I'm targeting 150mA. Using $V = RI$, we get
$R = (5\mathrm{V} - 1.4\mathrm{V} - 0.4\mathrm{V}) / 150\mathrm{mA} = 21.3\mathrm{\Omega}$.
This accounts for a 1.4V drop in the LED, and a 0.4V drop in the transistor.
I only had 39ohms resistors available, so I set two in parallel for $39/2 = 19.5=mathrm{\Omega}$ 
and a current of $164\mathrm{mA}$.

The current flowing out of the Raspberry Pi should be 16mA or less, and at the same time be large enough
to let current flow through the transistor. With $H_{FE}=100$, the current at the base needs to be at least 1/100th
the current flowing through the collector (164mA). So we need at least 1.6mA.
This time we have $(3.3\mathrm{V} - 0.75\mathrm{V}) / (1.6\mathrm{mA}) = R$ (the drop in the
transistor is 0.75\mathrm{V}). This gives us $R=1,593\mathrm{\Omega}$. I used a 1.5kOhms, so
the current at the base will be around 1.7mA.

## References

1. [LED datasheet](http://www.vishay.com/docs/81303/tshf5410.pdf)
2. How to read the datasheet/[Driving an Infrared Emitter in Steady and Pulsed Operating Modes](http://www.picbasic.co.uk/forum/attachment.php?attachmentid=7175&d=1386366870)
3. [Transistor datasheet](http://www.farnell.com/datasheets/661741.pdf)
4. https://blog.bschwind.com/2016/05/29/sending-infrared-commands-from-a-raspberry-pi-without-lirc/
5. [Maximum output current for the Raspberry
   Pi](http://www.thebox.myzen.co.uk/Raspberry/Understanding_Outputs.html)

Thanks to Vishay Semiconductor GmbH for pointing me to the second reference in this list!
