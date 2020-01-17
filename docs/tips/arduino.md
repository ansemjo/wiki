## Flashing over ISP header

Recently I had the need to program an Arduino that was not responding over USB,
i.e. the bootloader was probably broken somehow. I wrote a [blog entry][blog] about that.

[blog]: https://semjonov.de/post/2019-11/flash-arduino-without-a-bootloader-from-a-raspberry-pi/

The usual ISP header on an Arduino is mapped like this:

```
     ┏─────╮
MISO │ 1 2 │ VCC
 SCK │ 3 4 │ MOSI
 RST │ 5 6 │ GND
     ╰─────╯
```

### FTDI 232R Serial Chip

The FT232R provides a ["bit-bang" mode][bitbang] as well. Some breakout boards come with
an ISP header directly soldered on but you can also just use the breadboard pins on a full
breakout. I'm using a Sparkfun FT232R Breakout to do this.

[bitbang]: https://www.ftdichip.com/Support/Documents/AppNotes/AN_232R-01_Bit_Bang_Mode_Available_For_FT232R_and_Ft245R.pdf

These are the pins of a FT232R which correspond to the above ISP header:

```
     ┏─────╮
 CTS │ 1 2 │ VCC
 DSR │ 3 4 │ DCD
  RI │ 5 6 │ GND
     ╰─────╯
```

On the bottom of the Sparkfun breakout the legs are mapped like this:

```
         USB
┏───────────────────╮
│ ■ DCD     PWREN □ │
│ ■ DSR     TXDEN □ │
│ ■ GND     SLEEP □ │
│ ■ RI        CTS ■ │
│ □ RXD      V3.3 □ │
│ □ VCCIO     VCC ■ │
│ □ RTS     RXLED □ │
│ □ DTR     TXLED □ │
│ □ TXD       GND □ │
│      □ □ □ □      │
╰───────────────────╯
```

This programmer should come configured with a decently modern `avrdude` version already. If
it not here is a copy:

```
# see http://www.geocities.jp/arduino_diecimila/bootloader/index_en.html
# Note: pins are numbered from 1!
programmer
  id    = "arduino-ft232r";
  desc  = "Arduino: FT232R connected to ISP";
  type  = "ftdi_syncbb";
  connection_type = usb;
  miso  = 3;  # CTS X3(1)
  sck   = 5;  # DSR X3(2)
  mosi  = 6;  # DCD X3(3)
  reset = 7;  # RI  X3(4)
;
```

Using `avrdude` like this is said to be slower than other methods but in my testing it
turned out to be decently quick. Not "minutes" like some comments suggest anyway.

    avrdude -c arduino-ft232r -p m328p -v


### Raspberry Pi

At the time, I used the GPIO pins on a Raspberry Pi Zero W and amended the
`avrdude` configuration to use straightforward bit-banging. Here is a possible
mapping of the GPIO pins on the 40-pin header:

```
                15 ┆ · · ┆ 16
          3.3V  17 │ x · │ 18
(GPIO 10) MOSI  19 │ x x │ 20  GND
(GPIO 09) MISO  21 │ x x │ 22  Reset (GPIO 25)
(GPIO 11) SCLK  23 │ x · │ 24
                25 ┆ · · ┆ 25
```

This wiring can be used with the following `avrdude` programmer configuration:

```
# avr programmer via linux gpio pins
programmer
  id    = "gpio";
  desc  = "Use the Linux sysfs to bitbang GPIO lines";
  type  = "linuxgpio";
  reset = 25;
  sck   = 11;
  mosi  = 10;
  miso  = 9;
;
```

Put that in `~/.avrduderc` or a seperate file, which can be included with `avrdude -C +gpio.conf ...`.
Now use this programmer like this:

    avrdude -c gpio -p m1284p -v

