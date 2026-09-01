# Water Level Controller

Embedded control system on a PIC16F886 that runs a water pump from three level sensors, with a dry-run alarm and a pump run-time cutoff. Simulated in Proteus, then built on perfboard and tested driving a real aquarium pump.

What it does

Three sensors watch a reservoir, and the firmware drives two outputs — the pump and an alarm:

Sensor	Role
RA0	Low level — starts the pump
RA1	Dry-run detection — cuts the pump and raises the alarm
RA2	Alarm reset

Two protections sit on top of the basic fill loop:

Dry-run cutoff. If the dry-run sensor reads empty, the pump stops and the alarm latches. A pump running dry destroys itself; this is the reason the second sensor exists.
Run-time cutoff. If the level does not clear within the timeout, the pump shuts off anyway. This is the interlock that catches the case the sensors cannot see — a blocked inlet, a stuck float, a leak downstream.

The alarm latches rather than clearing itself. It stays raised until the reset sensor is triggered, so a fault cannot be silently absorbed while nobody is looking.

Hardware
PIC16F886, internal oscillator at 4 MHz
Comparators and analog inputs disabled — every pin used as digital I/O
Relay module switching the pump
Built on perfboard, not a fabricated PCB

The schematic in docs/ drives the relay with a BC548 transistor and a 1N4007 flyback diode — the textbook discrete driver. The build uses an off-the-shelf relay module, which integrates the same driver on its own board. The schematic and the photo differ for that reason; it was a deliberate substitution, not an error.

Firmware

Written in C for the XC8 compiler and built in MPLab.

firmware.c            complete firmware
docs/schematic-1.jpg  circuit, sheet 1
docs/schematic-2.jpg  circuit, sheet 2
docs/build-photo.jpg  assembled prototype
docs/report.pdf       written report (Portuguese)

Build with XC8 targeting PIC16F886, or open the folder as an MPLab X project.

Known defects

The code in this repository is the version submitted for assessment. Reading it back later turned up three real problems, all documented here rather than quietly patched — the history of the fixes is more useful than a clean repository.

The debounce function does not debounce.

c
unsigned char sensor1 = SENSOR1;                    // reads the pin once, into a local
unsigned char sensor1State = debounce(&sensor1);    // passes the address of that local

debounce() reads its pointer, waits 20 ms, then reads it again and compares. But both reads hit the same local variable, which nothing else can change during the delay — so the two reads always agree and the original value is returned unchanged. The delay happens; the debouncing does not. To work, the function has to re-read the port itself rather than a captured copy.

The pump timer runs about 20% slow.

bombaTimer += 50 assumes each loop iteration takes 50 ms. Each iteration calls debounce() three times at 20 ms each, so the real floor is 60 ms before any execution time is counted. The 5000 threshold therefore fires at roughly 6 seconds rather than 5. A hardware timer, or millis()-style elapsed-time arithmetic, is the correct approach — counting loop iterations and multiplying by an assumed constant is wrong by construction.

A stale comment. The comment above the timeout says 10 seconds while the constant says 5000. One of the two was changed and the other was not.

Context

Built in 2024 for the Microprocessor Systems subject of a Computer Engineering degree, by a team of three, supervised by Prof. Alexandre Dantas Dias.

It is an academic prototype driving a small aquarium pump — not a field-deployed product. It was never under version control at the time; this repository was created afterwards from the preserved firmware, schematics, photo and report.
