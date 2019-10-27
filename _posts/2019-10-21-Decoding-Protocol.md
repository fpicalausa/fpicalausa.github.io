---
layout: post
title: Controlling an Infrared LED - Air conditioner remote control
author: fpicalausa
---

This is the fifth article in a series about building a remote control for a air
conditioner over the internet. Check
[Part 1](/2019/10/06/Controlling-an-air-conditioner-remotely.html) for the
overview.

In the previous articles, we built a system that can both send and receive 
infrared signals. I will focus here on the general steps I took to decode the 
signal from pulses and spaces of various length to a meaningful signal.

## Extracting one's and zero's

As we saw in [part 3](/2019/10/06/Controlling-an-air-conditioner-remotely.html)
a common transport for remote controls is the pulse distance encoding: the distance 
(time elapsed) between two pulses determines whether the next bit is a 1 or a 0.
Using `mode2` from the LIRC package, we can easily analyse the timing between pulses.

Here is an example of a trace captured with mode2:

```txt
Using driver default on device /dev/lirc1
 16777215

     3361     1595      464      382      465      379
      464     1200      466      379      466     1199
      466      378      466      380      465      379
      465     1199      466     1199      465      380
      464      379      466      379      466     1199
      465     1199      465      379      466      379
      466      379      465      380      467      377
      465      380      464      379      466      379
      468      377      466      379      465      379
      466      380      465      380      464     1175
      489      379      465      377      468      380
      464      380      464      379      466      355
      489      379      466     1199      465      355
      489      380      465      380      465      381
      464      381      462     1199      466     1199
      466      379      465     1176      488     1174
      491      379      466    20261-pulse  2620267-space
 ```

Each row is a repetition of 3 pulse and space. As we can see, 
the pulses (first, third, and fifth columns) are almost all at around 
$465\mathrm{\mu s}$, except for the first ($3361\mathrm{\mu s}$),
and the last ($20261\mathrm{\mu s}$)) pulses. In-between pulses we
have spaces that are mostly around two values: $379\mathrm{\mu s}$,
and $1199\mathrm{\mu s}$. Again, the first and the last spaces
are different. Actually, we don't know for sure how long is the last 
space, since it corresponds by definition to no signal at all.

Let's arbitrarily assign bit value of 0 to spaces of $379\mathrm{\mu s}$, 
and bit value of 1 to spaces of $1199\mathrm{\mu s}$. This gives us roughly 
the following transport:
1. (header) Send a pulse of $3361\mathrm{\mu s}$, followed by a space of $1595\mathrm{\mu s}$,
1. (data) For each bit to transmit, send a pulse of $465\mathrm{\mu s}$ followed by:
   * a space of $379\mathrm{\mu s}$ for 0's,
   * a space of $1199\mathrm{\mu s}$ for 1's,
1. (trailer) Send a pulse of $3361\mathrm{\mu s}$, followed by a space of $1595\mathrm{\mu s}$.

The actual timing varies depending on the air conditioner, and
even for a given model there is some tolerance on the exact values. 
We can get more confidence in our understanding of the transport by capturing multiple different signals from
the remote.

With the transport figured out, we can decode the traces recorded by `mode2` into actual 0's and 1's to further understand what is being sent.
For example, running the above through [this python script](https://github.com/fpicalausa/remote-control/blob/master/tobits.py), we get the following:

```txt
1 00101000 11000110 00000000 00001000 00001000 01000000 
```

## Decoding Messages

Many air conditioners work by sending their complete state over the air on
every button press. That is, ever press of a button on the remote will send
out all the settings, including the current operation mode (heater, cooler,
etc.), the fan speed, the temperature, any timer set, etc. A few buttons can
additional send out short commands (turn off, set power-saving, etc.). Those
are general considerations to take into account when decoding the signal.

Because of their wide-spread use, there is a fair community around decoding
IR signals. Protocols are also often reused by the same makers. These are useful
resources even if one particular model hasn't been decoded yet.

For my Fujitsu AS-E4007T, here are the signal sent out by the remote control when turning the air conditioner on and off repeatedly:
```txt
1 00101000 11000110 00000000 00001000 00001000 01000000
2 00101000 11000110 00000000 00001000 00001000 00111111 00010000 00001100 10001110 10000000 00000000 00000000 00000000 00000000 01111010
3 00101000 11000110 00000000 00001000 00001000 01000000
4 00101000 11000110 00000000 00001000 00001000 00111111 00010000 00001100 10001110 10000000 00000000 00000000 00000000 00000000 01111010
```

We clearly see that there are two signal length: 1, and 3 correspond to short
commands; 2, and 4 correspond to the complete state of the air conditioner.
Short commands can usually be replayed as-is, since they do not contain
variable information. That is, a "turn off" command is always going to be a
"turn off" command.

The decoding is more complex for the state

## Controlling bits

Many of the commands on the air conditioner remote control have only a limited impact on the total state of the air conditioner. For instance, increasing the temperature by one degree at a time can help you isolate the bits that are responsible for the temperature.

These are traces captured when shifting the air conditioner from the minimal temperature of 20 degrees to 23 degrees:
```txt
1 00101000 11000110 00000000 00001000 00001000 00111111 00010000 00001100 00000010 10000000 00000000 00000000 00000000 00000000 11110001 
                                                                          ^^^^^^^^
2 00101000 11000110 00000000 00001000 00001000 00111111 00010000 00001100 00001010 10000000 00000000 00000000 00000000 00000000 11111110 
                                                                          ^^^^^^^^
3 00101000 11000110 00000000 00001000 00001000 00111111 00010000 00001100 00000110 10000000 00000000 00000000 00000000 00000000 11110110 
                                                                          ^^^^^^^^
```

Notice how one byte in the middle, and one byte at the end are changing.
Here, with some more observation, we can see that bytes are encoded as two
nibbles (4-bits), with the least significant bit first. That is, the value
1010 is decimal 5, followed by 0110 encoding decimal 6.

We can see that the last byte changes with any change in the message, by
recording other state. This byte is therefore likely a checksum, which
ensures the message integrity.

Certain bits stay at fixed values (perhaps used in some other models of the air conditioner), and can be replayed as is. Other are affected by the mode, fan speed,
timer.

## Checksum

IR Protocols often come with some form of error checking in the message. Some protocols send each byte followed by its logical inverse, while some other send a checksum along with the message.

For the Fujitsu AS-E4007T, the last byte is easy to identify as a checksum: any bit changing in the message will cause this byte to change as well. Comparing a sequences of messages where one 
byte value was increasing by 1 each time (the temperature), the checksum was instead decreasing by 1 each time. I made the assumption that it was the inverse of a checksum, summing each byte of the message, and them substracting them from 0. After some experimenting it turned out that only some of the bytes were summed up.

## Overal protocol

### Command

| Byte | Description |
|----|----|
| 0 - 4 | Magic value: `28 C6 00 08 08` |
| 5 | Command (see commands table) |
|| (End command if byte 5 is not `3F` ) |
| 6 - 7 | Magic value: `10 0C` |
| 8 | Power and temperature |
| 9 | Mode and timer mode |
| 10 | Fan speed and swing |
| 11 - 12 | Timer value |
| 13 | Checksum |

### Commands

| Value | Description |
|--|--|
| `40` | Off |
| `36` | Direction |
| `9C` | Dash |
| `90` | Eco-mode |
| `3F` | Long command |

### Power and temperature

Power and temperature are each encoded on 4 bits each.

| 0 - 3 | 4 - 7 |
|--|--|
| Power | Temperature |

| Power | Description |
|--|--|
| `1000` | Turn on (if the air conditioner was previously off) |
| `0000` | Stay on (if the air conditioner was previously on) |

The temperature is a 4-bit integer (LSB first) which represents an offset from 16 degrees celsius.

For example, the value `0010` = 4, represents 20 degree celsius.

The remote sets a maximum of 30 degrees (`0111`), and a minimum of 16 when heating, or 18 otherwise.

### Mode and Timer Mode

Mode and timer mode are each encoded on 4 bits each.

| 0 - 3 | 4 - 7 |
|--|--|
| Mode | Timer mode |

| Mode | Description |
|--|--|
| `0000` | Auto |
| `1000` | Cooler |
| `0100` | Dehumidifier |
| `1100` | Fan |
| `0010` | Heater |

| Timer Mode | Description |
|--|--|
| `0000` | Off |
| `0100` | Turn off after a certain time (see timer time) |
| `1100` | Turn on after a certain time (see timer time) |

### Fan speed and swings

Mode and timer mode are each encoded on 4 bits each.

| 0 - 3 | 4 - 7 |
|--|--|
| Fan speed | Swing |

| Fan speed | Description |
|--|--|
| `0000` | Auto |
| `1000` | High |
| `0100` | Low |
| `1100` | Quiet |
| `0010` | Natural |

| Swing Mode | Description |
|--|--|
| `0000` | Off |
| `1000` | On |

### Timer time

The timer time consists of two 12 bits values. The first one encodes the number of minutes
until turning the timer on, while the second encode the number of minutes until turning the
timer off. Which one is active depends on the timer mode value.

| 0 - 11 | 12 - 23 |
|--|--|
| Timer off | Timer on |

Both values are constructed in the same way. If the timer is set, the timer value consists
of a 10 bits integer LSB first indicating the number of minutes, followed by two bits at `01`.

For example, to start the air conditioner two hours later (120 minutes = `0001 111`), we set
the timer value to:

| 0 - 11 | 12 - 23 |
|--|--|
| `0000 0000 0000` | `0001 1110 0001` |

The timer mode in the command also needs to be set separately to `0011`.

### Checksum

The checksum is the difference from 0 from the LSB first integer value of bytes 7 to 12 in the command. Using properties of two's complement representation, with x the sum of bytes 7 to 12, we can also compute the result as: $\mathrm{result} = \neg x + 1$ (with $\neg$ the logical not operator).

## References

1. [Remote Central](http://www.remotecentral.com), a community of remote control enthusiast. In particular, [this post](http://www.remotecentral.com/cgi-bin/forums/viewpost.cgi?942295) was useful for me.
2. [IRremoteESP8266](https://github.com/crankyoldgit/IRremoteESP8266/tree/master/src), an arduino-based IR encoding library with implementation for many protocol.
3. [Two's complement](https://en.wikipedia.org/wiki/Two's_complement) properties.
