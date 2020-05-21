---
layout: post
title: Flashing USBaspLoader and QMK to an ATmega328P 
---

This is a super high level (and probably incomplete in some places) guide and list of resources related to using a Raspberry Pi's GPIO pins to flash USBaspLoader onto an ATmega328P microcontroller unit (MCU) and flashing QMK onto it over USB. I wrote it up after learning a bit about circuit design and MCU programming, and I hope it's helpful as a resource to others.

# Software prerequisites

You need to have avrdude and avr-gcc installed on a Raspberry Pi.

# Flashing USBaspLoader

These two sites were very helpful in wiring up the ATmega328P to the GPIO pins on the Raspberry Pi:

* [How to Program an AVR/Arduino using the Raspberry Pi GPIO](http://ozzmaker.com/program-avr-using-raspberry-pi-gpio/)
* [Program an AVR or Arduino Using Raspberry Pi GPIO](https://learn.adafruit.com/program-an-avr-or-arduino-using-raspberry-pi-gpio-pins/configuration)

The pin numbers in `avrdude.conf` are the GPIO numbers, **not** the physical pin numbers ([so with the pinout diagram here you use the gp* numbers in `avrdude.conf`, not the numbers on the pins themselves](https://www.jameco.com/Jameco/workshop/circuitnotes/raspberry-pi-circuit-note.html)).

Once everything is hooked up, make sure avrdude can see the MCU. This is assuming you named your custom programmer in `avrdude.conf` "pi".

```
avrdude -c pi -p m328p -v
```

Assuming that was successful, we can erase the MCU to start fresh:

```
avrdude -c pi -p m328p -v -e
```

You need to set various fuses on the MCU. [This is a super helpful beginner-level tutorial](https://medium.com/@MasonWallerQVJ/setting-up-an-external-crystal-clock-source-with-fuse-bits-for-avr-atmega-microcontrollers-31b2e79ae09a)

We'll set fuses for a 16 MHz crystal oscillator, unprogram CKDIV8 (divide clock speed by 8), disable brown-out detection, and probably some other things I'm forgetting about:

```
sudo avrdude -c pi -p m328p -v -u -U lfuse:w:0xFF:m
sudo avrdude -c pi -p m328p -v -u -U hfuse:w:0xD8:m
sudo avrdude -c pi -p m328p -v -u -U efuse:w:0xFF:m
```

[Download USBaspLoader here](https://github.com/baerwolf/USBaspLoader).

In `Makefile.inc` set:

```
F_CPU = 16000000
DEVICE = atmega328p
FLASHADDRESS = 0x7000
```

Compile and flash USBaspLoader:

```
make clean
make firmware
sudo make flash
```

Physically transfer the ATmega328P to another breadboard that's wired similar to [the `with-zener` schematic from the V-USB repo](https://github.com/obdev/v-usb/tree/master/circuits) (you can open this with an older free version of EAGLE; version 6.6.0 on macOS works) or the [schematic for the plaid keyboard](https://github.com/hsgw/plaid/tree/master/pcb) (you can open this with KiCad).

To test that the bootloader is working, hold down the switch that's wired to the `JUMPER_BIT` defined in USBaspLoader's `bootloaderconfig.h`, press the switch that's wired to the reset pin (pin 1 or PC6), release the reset switch, and then reset the jumper switch. This should boot the MCU into the bootloader that you just flashed, and you should be able to see the chip with avrdude:

```
avrdude -c usbasp -p m328p -v
```

If this doesn't work, something is probably wrong with your wiring or the fuses, or the bootloader didn't flash correctly.

# Flashing QMK

I don't have in-depth instructions for this, but the [QMK hand-wiring guide](https://docs.qmk.fm/#/hand_wire) is helpful. [All of the changes mentioned are in this commit](https://github.com/brianmutualaid/qmk_firmware/commit/f95689318fb97fdd4c6400d2c89bdd4ccfceae79).

[Clone the QMK repo here](https://github.com/qmk/qmk_firmware). Define the matrix rows and columns. Comment out soft serial pin. Add your layout to handwired2x2.h. Edit `rules.mk`:

```
MCU = atmega328p
ARCH = AVR
```

Comment out `BOOTLOADER`.

Add:

```
PROGRAM_CMD = avrdude -c usbasp -p m328p -v -U flash:w:$(BUILD_DIR)/$(TARGET).hex
```

Set `BOOTLOADER_SIZE` to `2048`.

Compile and flash from the QMK repo directory:

```
make handwired2x2:default:program
```

# Resources

* [ATmega328P datasheet](http://ww1.microchip.com/downloads/en/DeviceDoc/ATmega48A-PA-88A-PA-168A-PA-328-P-DS-DS40002061A.pdf)
* [USBaspLoader](https://github.com/baerwolf/USBaspLoader)
* [avrdude](https://www.nongnu.org/avrdude/)
* [QMK hand-wiring guide](https://docs.qmk.fm/#/hand_wire)
