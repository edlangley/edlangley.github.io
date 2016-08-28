---
layout: project_page
title: "PIC microcontroller ATA library"
date: 2015-01-25 21:43:40
permalink: /projects/picmicro-ata-lib/
description: "A library for interfacing with a PATA hard disc drive or CompactFlash with FAT32 read suppport on a PIC microcontroller, or other low RAM systems"
tags: embedded pic
---

This project started life a long time ago, with the intention to build an iPod clone, back when personal MP3 players were an expensive luxury and long before you could buy them from China on ebay for less than a light bulb. 

The plan for the MP3 player was to use a PIC microcontroller connected to a PATA hard disc drive (They were just called ATA back then, before SATA came along), the PIC would read MP3 files from a FAT32 partition on the drive and send them to an MP3 decoder chip through an audio DAC to earphones/speaker. A monochrome LCD with buttons would have provided the usual player controls and track selection, with a laptop hard drive giving plenty of song storage. I was going to mount all of this in a head unit in the dashboard of my car.

Unfortunately the project got shelved for about 10 years, until recently when I wanted my breadboard back. By now my car stereo has a USB socket to plug in mass storage and an SD card slot. So in order to bring this project to some sort of useful end, I finished off the library code I had written that provides access to files from a FAT32 filesystem on an ATA hard disc or CompactFlash with a PIC.

<!--more-->

So the ATA+FAT32 library is detailed here. It is a bit special, in that it is strongly suited to low RAM systems, and I mean looooowwwww RAM. I've had the ATA access code running on a PIC16F877 which only has 368 bytes of RAM. To get the FAT32 access in, I did have to go to a PIC18F452, which has about 1600 bytes or so of data RAM. If 368 bytes of RAM sounds like a typo (It isn't), bear in mind that PICs are based on a Harvard architecture, so the program memory, where the .text section of your program is stored, is counted seperately from the data RAM.

The astute reader will be wondering at this point how a 512 byte sector can be read from a hard disc into only 368 bytes of RAM. The answer is that it isn't. The API for the library is written in such a way that no complete sector is ever read at once. The caller of the library is able to set the LBA address for a read, the bytes of the sector are then read and examined, or skipped, as is needed by the calling code. Using this technique, even the FAT32 code, which provides read only access to files and directory entries, never needs to read a full sector.

**Get the source code:** from github [here](https://github.com/edlangley/PIC-ATA).

The rest of this page is extra detail, which may come in useful if you happen to decide to use the library in your own project.

## The Hardware

The circuit to connect the PIC micro to the ATA hard disc signals is based on a design used by Paul Stoffregen  in his [Using an IDE Hard Drive with a 8051 Board and 82C55 Chip](https://www.pjrc.com/tech/8051/ide/index.html) project.

I captured the schematic in Eagle format, available [here](https://www.dropbox.com/s/vm73l7ucejcwla5/pic-ata-eagle-schematic.zip?dl=1) and shown below.

<table id="captionedpicture">
	<tr><td>
		<img src="{{ site.url }}/img/projects/picmicro-ata-lib/pic-ata-schematic.jpg" alt="Schematic of PIC ATA circuit" />
	</td></tr>
</table>

As can be seen in the circuit, an [8255A](http://www.pci-card.com/8255.pdf) port expander is used to get the multitude of ATA signals down to a more sensible amount, in order to leave enough I/O on the micro to hopefully perform other functions. The 74HC04 inverters used on some of the ATA control lines between the 8255 and the drive are very much necessary. This is because when any of the 8255 port directions are changed via its one internal control register, all bits on the 3 ports are reset to zero. Some of the ATA signals are active low, so without the inversion, setting them low at the drive would cause an inadvertent read or write to the currently addressed ATA register in the drive, or reset the drive entirely.

Because I was holding off laying out a board until the other MP3 player components were added, the circuit was built up using jumper wires on a breadboard.

<table id="captionedpicture">
	<tr><td>
		<img src="{{ site.url }}/img/projects/picmicro-ata-lib/pic-ata-hw-full-overhead.jpg" alt="Photo of full hardware setup" />
	</td></tr>
	<tr><td>
		<img src="{{ site.url }}/img/projects/picmicro-ata-lib/pic-ata-hw-breadboard.jpg" alt="Photo of the breadboard wiring" />
	</td></tr>
	<tr><td>Top: Breadboard on the left with PIC ATA circuit+RS232 level shifting, ICD2 for programming+debugging at the top, on the right - a 10GB PATA hard disc drive.</td></tr>
</table>

## The Software

The library is written in C and can be built with either Microchips current XC8 compiler, or the now obsolete and quite old Hi-Tech PICC or PICC18 (I was using version 8, which is even older). Note that the code needs no changes between these two compiler suites apart from including a different header file. This is because Microchip bought Hi-Tech and if you look deeper into the header files, PICC is referenced a lot and the compilers are pretty much the same.

The ATA routines in the library are written for the 8255 circuit shown above, but there is some flexibility in which pins and ports are used on the PIC for the various signals. Simply change the #defines in the clearly commented sections within picata.c to account for any differences in pin assignment.

The ATA API described in picata.h provides the expected init routine, a get info function which reads off only the model/serial and drive geometry values, more as a check that the drive access is working than for any useful function, hence that can be disabled with a #define. As mentioned above, data access is performed by seperate API functions, one to set the LBA address of the required sector, another to read the next byte in the sector straight from the drive, and another to skip a given number of bytes in the sector.

Beneath the API functions, the next layer down deals with accessing ATA registers (See the ATA spec for details), and the bottom layer of functions deals with twiddling with the 8255 in order to waggle the ATA control signals on port C, whilst outputting register data or reading it back from ports A and B, being sure to reverse the port directions in the 8255 mode register at the appropriate points.

Initially, I had some 8255 port read/write routines which would automatically switch port directions in the control word as they went along. A single access to an ATA register requires numerous port accesses through the 8255, to toggle control and address signals to the drive on port C, read the register data through port A etc. Eventually I realised it was the 8255 mode register changes clearing all outputs, as explained above, which were negating the ATA chip select halfway through the ATA register access cycle.

Whilst trying to get the ATA register accesses working correctly, I wired up a logic analyser I had to hand, to see what the signals were doing.

<table id="captionedpicture">
	<tr><td>
		<img src="{{ site.url }}/img/projects/picmicro-ata-lib/pic-ata-hw-logicanalyser-overhead.jpg" alt="Photo of breadboard and logic analyser with its inputs connected to the PIC ATA circuit" />
	</td></tr>
	<tr><td>
		<img src="{{ site.url }}/img/projects/picmicro-ata-lib/logic-waveform-atactrl-output-changing-incorrectly.jpg" alt="" />
	</td></tr>
	<tr><td>Top: LogicPort USB logic analyser connected.<br />Bottom: Waveform captured showing unintended clearing of ATA control signals when attempting to read an ATA register from the drive.</td></tr>
</table>

The code was then re-written with the explicitly layered approach, the ATA register access routines now call down a layer to set the 8255 port directions at the beginning of the ATA access.

### Bad Signals

Once the waveforms for accesses of the low byte of the ATA data bus were looking good, I found that all the high bytes of the 16 bit data word accessed during sector data reads had consistent bit errors. Some of the logic analyser connections were moved to those data[8:15] signals, which then revealed a lot of ringing or oscillation of the logic level on one bit or another.

<table id="captionedpicture">
	<tr><td>
		<img src="{{ site.url }}/img/projects/picmicro-ata-lib/logic-waveform-data-line-noise.jpg" alt="" />
	</td></tr>
	<tr><td>Waveform captured showing noise on ATA data[9] signal.</td></tr>
</table>

This was solved in a highly technical fashion by having the demo code discussed below looping repeatedly on the PIC, then while that was gunning away, reaching over and poking the wires on the breadboard with my finger. With enough prodding suddenly good data would get read out. Must have been a poor ground or some crosstalk, after that every so often the IDE cable would need a good nudge, having discovered the sweet spot.

### Wiring Test

Whilst trying to diagnose the signal noise issue, I suspected there may be a wiring fault, and so a walking ones test for all pins on the 8255 ports was added. The logic analyser probes were then connected to the same rails as the ATA hard drive IDC header on the breadboard. It was only by moving the probe wires to the rest of the ATA data pins to see the test working, that the noise was caught.

<table id="captionedpicture">
	<tr><td>
		<img src="{{ site.url }}/img/projects/picmicro-ata-lib/logic-waveform-wiring-test-partial.jpg" alt="" />
	</td></tr>
	<tr><td>Waveform captured showing partial walking ones output.</td></tr>
</table>

### The Demo

There is some demonstration code in main.c, which does the following:

*Perform the wiring test if ATA_USE_WIRINGTEST is defined
*Print contents of drive sector 0
*Print contents of FAT32 BPB sector
*Read drive info if ATA_USE_ID is defined
 *Print all drive info bytes if HD_INFO_DBG is defined
 *Print a summary of drive model/serial/revision strings, drive geometry (Meaningless nowadays)
*Initialise FAT32 if FAT32_ENABLED is defined
 *Parse the entries of the root directory, printing the files and directories found
 *Open a file named HELLO.TXT if present, and print the contents

Here is the output from the demo code:

<div class="preformatted_console"><pre>
PIC ATA interface test
Init ATA: OK

Reading sector 0: LBA set

000:  33 C0 8E D0 BC 00 7C FB 50 07 50 1F FC BE 1B 7C
010:  BF 1B 06 50 57 B9 E5 01 F3 A4 CB BD BE 07 B1 04
020:  38 6E 00 7C 09 75 13 83 C5 10 E2 F4 CD 18 8B F5
030:  83 C6 10 49 74 19 38 2C 74 F6 A0 B5 07 B4 07 8B
040:  F0 AC 3C 00 74 FC BB 07 00 B4 0E CD 10 EB F2 88
050:  4E 10 E8 46 00 73 2A FE 46 10 80 7E 04 0B 74 0B
060:  80 7E 04 0C 74 05 A0 B6 07 75 D2 80 46 02 06 83
070:  46 08 06 83 56 0A 00 E8 21 00 73 05 A0 B6 07 EB
080:  BC 81 3E FE 7D 55 AA 74 0B 80 7E 10 00 74 C8 A0
090:  B7 07 EB A9 8B FC 1E 57 8B F5 CB BF 05 00 8A 56
0A0:  00 B4 08 CD 13 72 23 8A C1 24 3F 98 8A DE 8A FC
0B0:  43 F7 E3 8B D1 86 D6 B1 06 D2 EE 42 F7 E2 39 56
0C0:  0A 77 23 72 05 39 46 08 73 1C B8 01 02 BB 00 7C
0D0:  8B 4E 02 8B 56 00 CD 13 73 51 4F 74 4E 32 E4 8A
0E0:  56 00 CD 13 EB E4 8A 56 00 60 BB AA 55 B4 41 CD
0F0:  13 72 36 81 FB 55 AA 75 30 F6 C1 01 74 2B 61 60
100:  6A 00 6A 00 FF 76 0A FF 76 08 6A 00 68 00 7C 6A
110:  01 6A 10 B4 42 8B F4 CD 13 61 61 73 0E 4F 74 0B
120:  32 E4 8A 56 00 CD 13 EB D6 61 F9 C3 49 6E 76 61
130:  6C 69 64 20 70 61 72 74 69 74 69 6F 6E 20 74 61
140:  62 6C 65 00 45 72 72 6F 72 20 6C 6F 61 64 69 6E
150:  67 20 6F 70 65 72 61 74 69 6E 67 20 73 79 73 74
160:  65 6D 00 4D 69 73 73 69 6E 67 20 6F 70 65 72 61
170:  74 69 6E 67 20 73 79 73 74 65 6D 00 00 00 00 00
180:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
190:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
1A0:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
1B0:  00 00 00 00 00 2C 44 63 87 E9 00 F0 C9 02 00 01
1C0:  01 00 0C 3F E0 FF 20 00 00 00 E0 37 2A 01 00 00
1D0:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
1E0:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
1F0:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 55 AA

Reading sector 0x00004ABC: LBA set

000:  42 74 00 00 00 FF FF FF FF FF FF 0F 00 7A FF FF
010:  FF FF FF FF FF FF FF FF FF FF 00 00 FF FF FF FF
020:  01 66 00 69 00 72 00 73 00 74 00 0F 00 7A 5F 00
030:  66 00 69 00 6C 00 65 00 2E 00 00 00 74 00 78 00
040:  46 49 52 53 54 5F 7E 31 54 58 54 20 00 64 03 AB
050:  26 46 26 46 00 00 03 AB 26 46 00 00 00 00 00 00
060:  41 63 00 68 00 61 00 6E 00 6E 00 0F 00 69 65 00
070:  6C 00 73 00 2E 00 63 00 6F 00 00 00 6E 00 66 00
080:  43 48 41 4E 4E 45 7E 31 43 4F 4E 20 00 64 12 AB
090:  26 46 26 46 00 00 62 75 7D 45 03 00 46 09 00 00
0A0:  41 63 00 6F 00 72 00 65 00 00 00 0F 00 14 FF FF
0B0:  FF FF FF FF FF FF FF FF FF FF 00 00 FF FF FF FF
0C0:  43 4F 52 45 20 20 20 20 20 20 20 20 00 64 13 AB
0D0:  26 46 26 46 00 00 4F AD EE 44 04 00 00 10 95 01
0E0:  42 64 00 6C 00 69 00 6E 00 67 00 0F 00 D8 73 00
0F0:  2E 00 74 00 61 00 00 00 FF FF 00 00 FF FF FF FF
100:  01 74 00 75 00 72 00 74 00 6C 00 0F 00 D8 65 00
110:  61 00 72 00 74 00 5F 00 66 00 00 00 69 00 64 00
120:  54 55 52 54 4C 45 7E 31 54 41 20 20 00 64 13 AB
130:  26 46 26 46 00 00 43 A8 7E 43 AD 0C 57 06 00 00
140:  41 6D 00 69 00 6E 00 69 00 63 00 0F 00 25 6F 00
150:  6D 00 2E 00 6C 00 6F 00 67 00 00 00 00 00 FF FF
160:  4D 49 4E 49 43 4F 4D 20 4C 4F 47 20 00 00 02 9D
170:  2D 46 2D 46 00 00 18 72 4F 45 AE 0C D0 02 00 00
180:  42 79 00 00 00 FF FF FF FF FF FF 0F 00 DD FF FF
190:  FF FF FF FF FF FF FF FF FF FF 00 00 FF FF FF FF
1A0:  01 6B 00 69 00 63 00 61 00 64 00 0F 00 DD 2D 00
1B0:  65 00 6C 00 61 00 6E 00 67 00 00 00 6C 00 65 00
1C0:  4B 49 43 41 44 2D 7E 31 20 20 20 20 00 00 02 9D
1D0:  2D 46 2D 46 00 00 90 9A 71 45 AF 0C 06 00 00 00
1E0:  41 62 00 6C 00 61 00 68 00 2E 00 0F 00 9B 74 00
1F0:  78 00 74 00 00 00 FF FF FF FF 00 00 FF FF FF FF

Read drive info: 
000:  00 40 3F FF 00 00 00 10 00 00 00 00 00 3F 30 31
008:  31 32 00 00 59 38 30 33 45 33 37 41 20 20 20 20
010:  20 20 20 20 20 20 20 20 00 03 04 00 00 1D 53 41
018:  53 58 31 42 31 38 4D 61 78 74 6F 72 20 39 31 30
020:  30 30 44 38 20 20 20 20 20 20 20 20 20 20 20 20
028:  20 20 20 20 20 20 20 20 20 20 20 20 20 20 80 10
030:  00 00 2F 00 00 00 02 00 02 00 00 07 FF FF 00 01
038:  00 40 FF C0 00 3F 01 00 3C 20 01 2A 00 00 04 07
040:  00 03 00 78 00 78 00 78 00 78 00 00 00 00 00 00
048:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
050:  00 1E 00 17 7C 69 40 01 40 00 7C 69 40 01 40 00
058:  00 07 00 00 00 00 00 00 00 00 00 00 00 00 00 00
060:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
068:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
070:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
078:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
080:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
088:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
090:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
098:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0A0:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0A8:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0B0:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0B8:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0C0:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0C8:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0D0:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0D8:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0E0:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0E8:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0F0:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0F8:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
        Drive model: Maxtor 91000D8          
        Revision: SASX1B18
        Serial #:Y803
        Cylinders: 16383
        Heads: 16
        Sectors: 63

Mount FAT32: OK

Root dir contents:
FIRST_~1.TXT
CHANNE~1.CON
CORE        
TURTLE~1.TA 
MINICOM.LOG 
KICAD-~1    
BLAH.TXT    
TEST~1.JSO  
TEST.TXT    
TEST-F~1.TXT
SERIAL.C    
HELLO.TXT   
DATA            &lt;dir>
SAVE            &lt;dir>
PXTONE.DLL  
JELLY.EXE   
README.TXT  
TESTDIR         &lt;dir>
DOWNLO~1        &lt;dir>
LOGS            &lt;dir>
SETTIN~1.JSO
DATABASE    
FLATTR~1.CAC
GPODDER.NET 
FAT32.DEP   
FAT32.ERR   
FAT32.LST   
FAT32.SDB   
MAIN.DEP    
MAIN.ERR    
MAIN.LST    
MAIN.SDB    
MAIN(C~1.DEP
MAIN(C~1.LST
MAIN(C~1.SDB
NOTES.TXT   
PIC_ATA.ERR 
PIC_ATA.HEX 
PIC_ATA.PJT 
PIC_ATA.SYM 
SERIAL.DEP  
SERIAL.ERR  

Re-opening root dir: OK

Opening file HELLO.TXT: OK

Printing file HELLO.TXT:
Lorem ipsum dolor sit amet, consectetur adipiscing elit. Integer ultricies, diam at vestibulum interdum, sapien dolor
 feugiat arcu, consectetur dignissim odio sapien ornare nisl. Aenean auctor, eros tempus laoreet dignissim, eros tell
us porttitor est, eget consectetur elit felis vel metus. Sed congue vehicula lacus non malesuada. Vestibulum ac finib
us ligula, sit amet aliquet felis. Duis ac ex metus. Proin quis augue luctus, egestas magna ac, venenatis libero. Pra
esent tincidunt lectus dolor, id luctus nunc volutpat id. Nulla laoreet, ligula quis hendrerit iaculis, nunc velit fa
cilisis purus, in malesuada ligula odio ac sapien. Proin blandit erat lacus, sed posuere diam pharetra nec. Nunc susc
ipit tristique vehicula. Morbi sem libero, semper id sollicitudin in, sagittis eu odio.

Suspendisse volutpat urna arcu, in malesuada urna suscipit id. Phasellus eu lectus non nunc dignissim condimentum pha
retra vel metus. Vivamus vehicula interdum mi eget bibendum. In sed interdum mi, sit amet lacinia ante. Nam dignissim
 nunc sit amet porta consequat. Vivamus maximus dolor ex, eget ullamcorper massa dictum ac. In quis odio at lorem lac
inia fermentum at quis eros. In non ultricies nunc, id convallis erat. Sed tellus urna, consectetur ut gravida a, ali
quet id lectus.

Duis in ultrices tortor. Vivamus tempor est at pharetra molestie. Quisque eu semper mi. Integer et finibus magna. Fus
ce auctor ipsum sit amet nunc ultricies, quis ultricies metus mattis. Sed ante purus, pulvinar quis lacus eget, pelle
ntesque scelerisque orci. Aliquam tristique sit amet justo eu suscipit. Maecenas ac tortor eget justo pretium tincidu
nt ut ac dui. Cras pretium risus sed posuere scelerisque. Proin facilisis blandit ornare. Etiam at lorem et mauris ef
ficitur tempor efficitur sit amet metus. Sed vel placerat elit. Cras consequat quam eu arcu luctus, vitae consectetur
 est pellentesque. Nulla facilisi.

</pre></div>

## Leftover MP3 stuff

So what came of the rest of the MP3 player you ask? Well the decoder chip was to be an STMicroelectronics STA015 of which I obtained a few samples in the QFP package. That would output digital audio samples in a serial stream to a Cirrus Logic CS4334 sterio audio DAC, then from there out to the speaker connections. I had extended the [ATA circuit schematic with the extra MP3 components](https://www.dropbox.com/s/0838ueavwu8gn4s/pic-mp3-eagle-schematic.zip?dl=1). Be warned it's not tested though.

The decoder appears to be long since out of production. However, given that all this was in the days before breakout boards all over Sparkfun and Adafruit, the lesson is: never let anyone tell you that you can't get a fine pitch surface mount package soldered down onto a home made PCB.

<table id="captionedpicture">
	<tr><td>
		<img src="{{ site.url }}/img/projects/picmicro-ata-lib/QFP-SO-to-DIL-adapters.jpg" alt="Photo of home made PCBs with QFP and SO-8 chips soldered to them" />
	</td></tr>
</table>

Schematics for those very simple adapter boards are [here](https://www.dropbox.com/s/cpn23vssehzodxh/SMT-QFP-SO8-SO28-to-DIL-eagle-schematics.zip?dl=1).





