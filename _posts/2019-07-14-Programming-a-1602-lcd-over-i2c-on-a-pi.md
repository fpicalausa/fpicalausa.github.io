---
layout: post
title: Programming a 1602 LCD over I2C on a Raspberry Pi
author: fpicalausa
---

I recently acquired a Raspberry Pi, along with a bunch of electronic bits and 
pieces in the hope of getting back to tinkering. My last experience with 
electronics was about 9 years ago. While the basics haven't changes much,
my kit came with just the bare minimum information to make things working:
connect this like that, run this python script, see that the LCD shows stuff. 
This isn't quite enough to satisfy my curiosity.

After looking up for information on the Internet, I found that: 
- the python code I was given was mostly copy-pasted to death. It looks
  something like this 

    ```python
	import smbus
	import time
	 
	# Define device parameters
	I2C_ADDR = 0x27 # I2C device address, if any error, 
	# change this address to 0x3f
	LCD_WIDTH = 16 # Maximum characters per line
	 
	# Define device constants
	LCD_CHR = 1 # Mode - Sending data
	LCD_CMD = 0 # Mode - Sending command
	 
	...

	# Initialise display
	def lcd_init():
	    lcd_byte(0x33,LCD_CMD) # 110011 Initialise
	    lcd_byte(0x32,LCD_CMD) # 110010 Initialise
        lcd_byte(0x06,LCD_CMD) # 000110 Cursor move direction
        lcd_byte(0x0C,LCD_CMD) # 001100 Display On,Cursor Off, Blink Off 
        lcd_byte(0x28,LCD_CMD) # 101000 Data length, number of lines, font size
        ...
    ```

- there is some simpler code using
  [`LiquidCrystal_I2C.h`](https://github.com/fdebrabander/Arduino-LiquidCrystal-I2C-library)
  in C, or
  [i2c-lcd](https://i2c-lcd.readthedocs.io/en/latest/usage.html in python)

- there are tools for generting custom characters, (like
  [this one](https://maxpromer.github.io/LCD-Character-Creator/)), but they
  mostly speak c.

- some people wire the lcd directly to their arduino/raspberry pi (with anywhere
  from 5 to 16 pins), while some other use a "backpack" i2c adapter (with just 
  4 pins used).

Unfortunately, there is very little explanation on what the commands are, why
we need smbus, why we'd want to wire the lcd direct or use an adapter, how to
use custom characters with python, or generally how it all works. So here's what
I found researching this topic. This is somewhat specific to the kit I bought,
so I'll try to explain what I have so that you can adapt it to your setup.

# The physical layer

The kit I'm using includes a Raspeberry pi 3 b+, a lcd screen board with a
HD44780 compatible controller, as well as a PCF8574 chip that converts serial
i2c signals into 8-bits parallel output (it can also convert parallel input into
an i2c signal). The PCF8574 is fitted onto a board (a "backpack" board), that
connects specifically to the lcd board, providing inputs for some of the 16 
inputs.

The reason for putting a PCF8574 in front of any other component is that it
usually takes less pins from the controller (arduino or raspberry pi),
than by connecting it directly. Here, we can connect the PCF8574 to the
raspberry pi with only 4 pins, instead of occupying 16 pins. The tradeoff here
is that we need to follow the i2c protocol, and have an added translation step.
Fortunately, the rasbperry pi comes with good support for i2c so there isn't
much of an overhead.

## HD44780 board

Let's now consider what pins are on each of these boards. For the HD44780 board,
the pinout looks like this:

| Pin number | Name | Description |
|-|-|-|
| 1 | VSS  | Negative supply voltage |
| 2 | VDD  | Positive supply voltage (mine require +5V) |
| 3 | VO   | Contrast adjustment (this is typically linked to a potentiometer, which happens to be on the backpack board) |
| 4 | RS   | Register select (0 for an instruction, or 1 for data) |
| 5 | RW  | Read/Write (0 for write, or 1 for read) |
| 6 | E    | Enable (triggered by a falling edge)
| 7...14 | D7...D4 | Data bus (there is a 4 bits mode where only D7 to D4 are used)  |
| 15 | A | backlight anode |
| 16 | K | backlight cathode |

Aside from the usual technical (voltage supply, contract, etc.), we can already
see that we have a byte sized bus, along with RS/RW/E to control what operation
is being fed when.

Reading through the tech sheet of the HD44780, you will find that it has 3
separated memory chips. One ROM with the fixed character shapes (CGROM), 
one that holds custom characters shapes (CGRAM), and the other that
contains the current display of the lcd (DDRAM).

## PCF8574 board

For the PCF8574 board, the pinout for i2c is much simpler:

| Pin number | Name | Description |
|-|-|-|
| 1 | GND  | Ground|
| 2 | VCC  | Positive supply voltage (mine require +5V) |
| 3 | SDA  | Data line |
| 4 | SCL  | Clock line |

The pinout for the parallel consists is 16 unnamed pins. 

The PCF8574 chip itself has 16 pins as well. Without diving into the details, it uses 
- 4 pins from the board for interfacing with the i2c protocol (gnc, vcc, sda,
  clk) directly on the board, 
- 3 pins from an address selector (A0...A2) that defines the address used when
  identifying itself in the i2c protocol,
- 8 i/o ports (P0...P7) for the parallel interface 
- 1 pin is unused.

For driving our lcd, what happens is essentially that with just a clock and data
line, the PCF8574 stores individual bits from the data line in sequence, and
then replicating their values on its 8 i/o ports.

Now, perhaps surpringly the P0...P7 pins aren't connected to the D0...D7 input
of the LCD. Instead, (TODO: check this) P0...P2 are connected to the RS, RW,
and E input of the lcd board to signal whether we want to read/write data or
instructions from our HD44780, and then D4...D7 are fed with the right data.
We're operating in 4 bits mode, so there is no need for D0...D3, and those are
set to ground.

## Raspberry pi

The raspberry pi pinout has a lot going, but we only need 4 what is required for
i2c communication:

| Pin number | Name | Description |
|-|-|-|
| 2 | 5V Power | |
| 3 | SDA / BCM2  | Data line |
| 5 | SCL / BCM3  | Clock line |
| 6 | Ground   | |

You will need to enable I2C through `raspi-config` to be able to use the i2c
interface.

# Communication layer: I2C and smbus

From the above, we know we'll need to send some sequence of 7bits through I2C to 
send the right values of RS/RW/E/D7...D4 down to the LCD.

I2C is a fairly complex protocol, designed to inter-connect different integrated
circuit (hence the name inter-integrated circuits, or I2C for short). It was
invented in 1982, became free of licensing fees in 2006, and is still in use
today. For a single master system, the basics of it are as follows:

- The protocol only uses a serial data (SDA), and a serial clock wire (SCL) The 
  master sets a clock signal on the clock wire.
- The protocol serves as a data bus, allowing both master and slaves to transmit
  and receive information
- One bit of data is send on every clock tick; Some special start and stop
  commands are identified by keeping the clock on hi for one tick, and changing
  the data signal. 
- Start and stop commands allow the slave to see that it should start listening
  for some transmission.
- there is a ACK (or NACK) signal after each byte of transmission.

When a master device wants to write to a slave device, here is what happens
- the master sends out a START command
- the master sends out a 7 bits address
- the master send a read/write bit with value 0 indicating it want to write. 
- the slave with the corresponding address sends out a ACK signal.
- the master sends out a byte of data which the client ACK or NACK.
- this repeats for so long has the master needs to transmit data.
- the master finally sends out a STOP command to end the transmission.

In short:

```
    <START><7 bit Addr><R/W><slave ACK><8 bit data><slave AKC>...<STOP>
```

See figure 9 and 11 of the 
[I2C specification](https://www.nxp.com/docs/en/user-guide/UM10204.pdf) for 
details.

A note on smbus:
Smbus is a protocol built on top of i2c, preserving the transmission format, but
adding semantics to some bytes.

Essentially:
- The first byte of data written by a master to a slave is a command. The slave
  can use all or part of this byte to perform the required function.
- Optionally, the master can follow with more data bytes. These are interpreted
  by the slave in the context of the command.

NOTE: this means that sending a single command through smbus means that we
are effectively sending one byte to an i2c the slave.

## HD44780 protocol

The HD44780 chip itself is used for different LCD screens (say, of different
size), as well as different ways of interfacing externally (say, 4bits or 8
bits, different speed, etc.). The chip itself doesn't know its environment at
first and hence needs to be initialized properly. This is done by sending a
sequence of instructions telling the chip what to expect.

### Sending an instruction

So, how do we send an instruction? The datasheet lists all possible instructions
on page 23, and explains timing on p52.

For display clear, in 8-bit mode, the
instruction looks as follows:

| RS | RW | D7...D0 |
|-|-|-|
| 0  | 0  | 00000001|

We need to turn the enable signal on and off for the chip to recognize it, so it
looks a little more like this.
<table>
  <thead>
    <tr>
      <th>time</th>
      <th>RS</th>
      <th>RW</th>
      <th>D7…D0</th>
      <th>E</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>00000001</td>
      <td>1</td>
    </tr>
    <tr>
      <td colspan="5" style="text-align:center">[hold]</td>
    </tr>
    <tr>
      <td>230ns</td>
      <td>0</td>
      <td>0</td>
      <td>00000001</td>
      <td>0</td>
    </tr>
    <tr>
      <td colspan="5" style="text-align:center">[hold]</td>
    </tr>
    <tr>
      <td>500ns</td>
      <td>0</td>
      <td>0</td>
      <td>00000001</td>
      <td>0</td>
    </tr>
  </tbody>
</table>

The timing here correspond to the values in the datasheet (p 52 and fig 25).
After reading an instruction in, the controller will set the busy flag to 1. We
have to wait for this busy flag to be back to zero; this can be done either by
polling D7 which replicates the busy flag, or by waiting for more than the
execution time of the instruction (Table 6 in the datasheet).
This comes to roughly 1.6ms for clear display/cursor home instructions, and
40μs for all other commands. Of course, with replica chips, the timing can vary, 
but these seems to be a widely used approximations.

Building back on our previous example:

| time | RS | RW | D7...D0 | E |
|-|-|-|-|-|
|  0   | 0  | 0  | 00000001| 1 |
| \[hold\]   | 0  | 0  | 00000001| 1 |
| 230ns | 0  | 0  | 00000001| 0 |
| \[hold\]   | 0  | 0  | 00000001| 0 |
| 500ns | 0  | 0  | 00000001| 0 |
| \[wait 1.64ms\]   | 0  | 0  | 00000001| 0 |

### Sending an instruction (4 bits mode)

Since we work in 4 bits mode, the HD44780 datasheet tells us that we need to
split our data bytes into two 4 bits nibbles and keep the RS and R/W signals.

Coming back to the previous example, the clear display instruction becomes

| time | RS | RW | D7...D4 | E |
|-|-|-|-|
|  0   | 0  | 0  | 0000 | 1 |
| \[hold\]   | 0  | 0  | 0000| 1 |
| 230ns | 0  | 0  | 0000 | 0 |
| \[hold\]   | 0  | 0  | 0000 | 0 |
| 500ns | 0  | 0  | 0001| 1 |
| \[hold\]   | 0  | 0  | 0001| 1 |
| 730ns | 0  | 0  | 0001 | 0 |
| \[hold\]   | 0  | 0  | 0001 | 0 |
| 1000ns | 0  | 0  | 0001 | 0 |
| \[wait 1.64ms\]   | 0  | 0  | 0001| 0 |

Other instructions are split similarly. Figure 17 of the datasheet shows how to
check the busy flag.

### Initializing the controller

The datasheet explains that there are two modes of initialization, through 
an internal reset circuit and an external one. The internal reset sets
a 8bit interface, 1 line display. Our LCD is 2 lines and we use a 4 bits
interface, so we'll instead need to follow the 4-bit initialization procedure on 
page 46.

Here are the main steps:

| RS | RW | D7...D4 (1) | D7...D4 (2) | Description  |
|-|-|-|-|
| 0  | 0  | 0011    |  | Function set, 8bit, (8 bits instruction) |
| 0  | 0  | 0011    |  | Function set, 8bit, (8 bits instruction) |
| 0  | 0  | 0011    |  | Function set, 8bit, (8 bits instruction) |
| 0  | 0  | 0010    |  | Function set, 4bit, (8 bits instruction, after this we are ready to send 4 bits instructions) |
| 0  | 0  | 0010    | 1000 | Function Set, 4 bits, 2 lines, 5x8 font | 
| 0  | 0  | 0000    | 1000 | Display off |
| 0  | 0  | 0000    | 0001 | Display clear |
| 0  | 0  | 0000    | 0110 | Entry mode (cursor direction: increment, and no display shift (0) |

# Writing a python library


To tie this all up, I wanted to write a simple python library that helps with 
manipulating my lcd.

But first of all, let's get back to the first sample of code to understand how
it works.

## Checking the existing code

The existing copy-pasted code relies heavily on a `lcd_byte` function, that
sends data through smbus. Since it is i2c compatible, using the python 
`write_byte` method on a SMBus object means that we'll write bytes to our 
PCF8574, which in turns writes to the parallel input of our LCD.

Checking the `lcd_byte` function, we see that each byte has the following
layout:

| | 0 ... 3 | 4 | 5 | 6 | 7 | 8 |
|-|-|-|-|-|-|-|-|-|
| Python | `bits_high`/`bits_low` | `LCD_BACKLIGHT` | `ENABLE` | n/a | `LCD_CMD`/`LCD_CHR` |
| LCD | D7...D4 | ? | E | RW | RS |

The `lcd_byte` function starts by splitting the 8-bits instruction into two
nibbles, combining each of them with `LCD_BACKLIGHT` and `LCD_CMD`/`LCD_CHR`
inputs into the `bits_high` and `bits_low` byte-sized variables. It then
proceeds to send out `bits_high` on the bus, keeping Enable low. This is likely
to ensure that the data we end up sending is stable by the time we toggle
Enable (from the PCF8574 datasheet, we can see that it takes 4 microseconds for
the data to be valid after acknowledgment). 

`lcd_byte` then proceeds to set Enable (sending the whole `bits_high` with the
enable bit active), wait for a certain amount of time, disable Enable, and wait
some more. After this is sends out the `bits_low` over the bus in the same way.

The initialization procedure is as follows:

  ```python
    lcd_byte(0x33,LCD_CMD) # 110011 Initialise
    lcd_byte(0x32,LCD_CMD) # 110010 Initialise
    lcd_byte(0x06,LCD_CMD) # 000110 Cursor move direction
    lcd_byte(0x0C,LCD_CMD) # 001100 Display On,Cursor Off, Blink Off 
    lcd_byte(0x28,LCD_CMD) # 101000 Data length, number of lines, font size
  ```

This roughly matches with the initialization procedure from the datasheet:
1. it starts sending out the first two function set in the first line. Note that
   `lcd_byte` still split 0x33 into two nibbles, but each one of them is
   interpreted as a 8 bits instruction.
2. it then sends out the second two function set in the second line as 8 bits
   instruction. We're now in 4 bits mode.
3. it continues by setting the entry mode (3rd line) and the display settings
   (4th line). 
3. it sends out the function set for 4bits, 2 lines, 5x8 font.


## A new library

CAUTION: I only tested this with my kit, and I've just learned the stuff so
use at your own risk.

To put all the things I learned into practice, I wrote a small library that 
illustrates my learnings. You can find it in my 
[HD44780-over-smbus](https://github.com/fpicalausa/HD44780-over-smbus)
repository.

The included example illustrates how to show custom characters as well: use 
the set CGRAM address instruction to select character memory, start writing
the character paying attention to the necessary padding, switch back to DDRAM 
addresses, and write the corresponding character. The HD44780 datasheet is again 
helpful for this.

-------------------------------------------------------------------------------

References

HD44780

- http://elm-chan.org/docs/lcd/hd44780_e.html
- [Hitachi datasheet](https://www.sparkfun.com/datasheets/LCD/HD44780.pdf)
- http://www.epemag.wimborne.co.uk/resources.htm
- [Entry modes](http://www.handsonembedded.com/lcd16x2-hd44780-tutorial-4/)

PCF8574

- https://simple-circuit.com/arduino-i2c-lcd-pcf8574/
- https://www.nxp.com/docs/en/data-sheet/PCF8574_PCF8574A.pdf

Rapberry pi

- https://pinout.xyz/

I2C

- https://en.wikipedia.org/wiki/I%C2%B2C
- https://www.nxp.com/docs/en/user-guide/UM10204.pdf

SMBUS
- http://smbus.org/specs/SMBus_3_0_20141220.pdf
- [Python smbus module](https://i2c.wiki.kernel.org/index.php/I2C_Tools)
