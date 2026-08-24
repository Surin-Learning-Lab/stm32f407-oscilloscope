# Engineering log

## 2026-08-23
Repo created. Nothing built yet.

The board has no dedicated SWD header, only the 20-pin JTAG/SWD connector (P1), and it isn't silkscreened. From the side furthest from the SD card mount, the bottom row is GND except pin 1(3.3v). pin4 is SWDIO and pin5 is SWCLK, use any pin on the bottom for GND except for pin1.