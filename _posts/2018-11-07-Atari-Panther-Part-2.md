---
layout: post
title: The Atari Panther - Part 2 - The hardware
---

In this second part we explore the Atari Panther from an hardware point of view.

<table style="width:75%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img src="{{ site.url }}/images/atari-panther-2/image_0.png"></td>
  </tr>
  <tr>
    <td>A <a href="https://imgur.com/a/lbdFc">3D rendering</a> of the Atari Panther</td>
  </tr>
</table>

Articles:
* [Part 1 - The history](2018-10-07-Atari-Panther-Part-1.md)
* Part 2 - The hardware

# The sources

The Atari Panther was never produced, anyway developer kits were produced and relevant documents emerged over time. These are the sources for this series of articles.

* [Documents](http://www.atarimuseum.com/videogames/consoles/jaguar/Panther/index.htm) at Atari Museum
* [Schematics](https://www.chzsoft.de/asic-web/console.pdf) of the console and developer kit recovered by Christian Zietz
* Photos of the [developer kit](http://www.homecomputer.de/pages/panther.html)
* The [developer manuals](http://www.atarimuseum.com/videogames/consoles/jaguar/Panther/PantherHW%20DocumentsFlare%20II.zip)

# Overview

The console is a cartridge based system. The cartridge is inserted on the front like in the Nintendo NES.

<table style="width:50%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img style="vertical-align:middle;" src="{{ site.url }}/images/atari-panther-2/panther-front.jpg"></td>
  </tr>
  <tr>
    <td>Front <a href="https://imgur.com/a/lbdFc">rendering</a> of the Atari Panther</td>
  </tr>
</table>

Two joystick ports are available on the front. The joystick ports have 15 pin and are compatible with the Atari Jaguar and some Atari home computers.

<table style="width:50%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img style="vertical-align:middle;" src="{{ site.url }}/images/atari-panther-2/panther-back.jpg"></td>
  </tr>
  <tr>
    <td>Back <a href="https://imgur.com/a/lbdFc">rendering</a> of the Atari Panther</td>
  </tr>
</table>

An expansion port is available on the back together with RF, composite and L/R audio connectors.

# Architecture

The machine seems designed around the idea of a single 8 Mhz 32 bit bus connecting all the components and a small amount of 32 bit SRAM. This simplicity results in a cheaper motherboard compared to the ones of the competitors where multiple buses and RAM subsystems are used.

```
                                            a
  ┌─────┬─────┬─────┬─────┬─────┬───────────┬─────────────┐
  │     │     │     │     │     │    p      │      ld     │
[CTG] [EXP]  [AC] [ROM] [CPU]  [GS]═════[Panther]──────[lSRAM]
  │     │     │     │     │     │           │      ud     │
  └─────┴─────┴─────┴─────┴─────┴───────────┴──────────[uSRAM]

a       = address bus
ld      = lower data bus
ud      = upper data bus
p       = pixel bus

lSRAM   = lower SRAM            ROM  = Boot ROM
uSRAM   = upper SRAM            AC   = Audio Chip
Panther = Panther Chip          EXP  = Expansion Port
GS      = GameShifter Chip      CTG  = Cartridge Port
CPU     = CPU
```

<table style="width:50%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img style="vertical-align:middle;" src="{{ site.url }}/images/atari-panther-2/components.jpg"></td>
  </tr>
  <tr>
    <td>Position of the components on the Developer Kit motherboard.</td>
  </tr>
</table>

The main bus is composed of a 24 bit address bus (a), a 16 bit upper data bus (ud) and a 16 bit lower data bus (ld). This allows to connect some components through a 32 bit data path and some others through a 16 bit data path.

The Panther Chip has a 32 bit data path being the only component connected to both data buses. The CPU and all the other components have a 16 bit data path being connected to the upper data bus.

The 32 KB of SRAM are organized as two 16 bit banks, one connected to the upper 16 bit data bus (uSRAM) and the other to the lower 16 bit data bus (lSRAM). Both banks are connected to the address bus.

The Panther Chip works as a de-multiplexer connecting the correct SRAM bank based on the selected address. Even words are connected to the upper bank directly.

```
                             a
...─┬───...──────────┬─────────────┐
    │                │             │
  [CPU]          [Panther]...      |
    │                │      ud     │
...─┴───...──────────┴──────────[uSRAM]
```

Odd words are connected to the lower bank through the Panther Chip and another 16 bit data bus (ld).
```
                             a
...─┬──...───────────┬─────────────┐
    │                │      ld     │
  [CPU]          [Panther]──────[lSRAM]
    │                │             :
...─┴──...───────────┴...
```

We can say that the Panther is a hybrid 16/32 bit machine.

# RAM and ROM

The main SRAM amounts to just 32 KB. It is organized in two 16 bit banks. Each bank is composed of two chips with 8K x 8 bit.

Since each chips can be individually enabled it is possible to read/write just some of the bytes during an access.
This permits byte accesses but it also permits to mask some bytes during a 32 bit write access.

For example if the destination long word contains 0xAABBCCDD, the data bus contains 0x00112233 and the 3rd chip is not selected then the 3rd byte is masked.

    0xAABBCCDD : SRAM content
    0x00112233 : write data
      eeee--ee : enable mask
    0x0011CC33 : new SRAM content

As far I know this feature is not used on the Panther.

The main SRAM run @ 8 Mhz (125ns, 2 mclk).
<br>*NOTE*: a mclk is a 16 Mhz cycle, 62,5 ns.

The Panther contains also 64KB of 16 bit bootstrap ROM. It is composed of two 32K x 8 bit chips running @ 8 Mhz (125ns, 2 mclk).

# Cartridge and Expansion ports

The Cartridge port has a 16 bit data bus (ud) and a 24 bit address bus (a).
It supports:
* Up to 6MB of ROM @ 4 Mhz (250ns, 4 mclk)
* Up to 32KB of RAM @ 4 Mhz (250ns, 4 mclk)

This means a bandwidth of 8 MB/s.

The Expansion port has a 16 bit data bus (ud) and a 24 bit address bus (a). Is not clear what is the access speed.

# CPU

The CPU is a Motorola 68000 @ 16 Mhz. It can theoretically [reach](https://en.wikipedia.org/wiki/Instructions_per_second#MIPS) 2,8 MIPS.

Thanks to the main bus, the CPU can access all the other components in the system. This is different from other consoles where different buses exists and the CPU cannot access them directly.

The 68000 requires 2 mclk to prepare the access and then it waits 2 mclk for the access. If the accessed memory requires more than 2 mclk then the 68000 has to be slowed down inserting wait states.

```
    __AA__AA__AA...
       ^   ^   ^
    standard read access
```

All the components in the system have a 2 mclk access cycle except the Cartridge RAM and ROM which need 4 mclk. This means that the 68000 has no wait states except when accessing the Cartridge RAM/ROM. In this case 2 wait states are introduced.

When the CPU is executing code from Cartridge ROM it requires 6 mclk to fetch 1 instruction instead of 4 mclk. It means that is 50% slower than when executing code from main SRAM. Copy critical code in SRAM is an evident optimization.

```
    __AAAA__AAAA__AAAA...
         ^     ^     ^
    ROM read access
```

# The Panther Chip

The Panther Chip (PC) is the "heart" of the system and performs these functions:
* Support logic for the CPU
* Bus arbiter for the main bus
* Address decoding
* Bridge for the lower data bus
* Generator of clocks and video synchronization signals
* Contain the Object Processor
* Feed the GameShifter chip via a 5 bit data bus and two 9 bit address bus.

On the Developer Kit the Panther Chip has 160 pins, works @ 16 Mhz and is implemented on a [Toshiba TC110G32](https://datasheetspdf.com/mobile-pdf/905878/Toshiba/TC110G32.html). This is a Gate Array not a custom ASIC. It is not clear if actual ASICs were planned or ever produced. The chip contains 32K gates (13K usable according to the datasheet). By contrast the Amiga Agnus chip is composed of about [7K gates](https://retrocomputing.stackexchange.com/questions/3000/how-many-gates-does-each-chip-in-ocs-have?rq=1). The gate array has a gates with 0.6 ns delay, this means that at 16 Mhz (62.5 ns) pipeline stages are max 10 gates deep.

# The Object Processor

The Object Processor inside the Panther Chip is where the graphic is generated. To understand how it works we can draw some similarities with how modern 3D graphics card works.

In a 3D card, every frame, a GPU executes a list of commands (like draw a triangle, change texture, etc...) and draws in a dedicated high speed frame buffer.

The Object Processor is similar. It draws rectangles instead of triangles, the buffer is just one scanline and the list of commands (Object List, OL) is execute at every scanline instead of every frame. The line buffer is actually on the GameShifter Chip.

There is no limit to the Objects into the OL except the SRAM size and the time available to process it (a scan line, 64 usec).

The Object Processor can fetch graphic directly from the ROM. This is another important difference compared to Super NES and Megadrive. These machines can fetch graphic only from dedicated graphic RAM and therefore are continuously streaming data from ROM to the graphic RAM. In this regard the Panther resembles the SNK Neo Geo.

# The GameShifter Chip

The GameShifter chip (GS) has 84 pins and works @ 16 Mhz. The  [schematics](https://www.chzsoft.de/asic-web/console.pdf) show that it contains:
* Two line buffers (a front and a back buffer)
  * The line buffers have 320 pixel with 5 bit depth (2^5=32 colors).
  * The front buffer can be copied in a single clock cycle on the back buffer.
  * The front buffer can be only written (not read) by:
    * the Panther Chip @ 16 Mhz (62 ns, 125 ns).
    * the CPU @ 8 Mhz (125 ns, 2 mclk).
* a Color Look Up Table (CLUT or Palette)
  * has 32 colors with 18 bit depth (262K shades).
  * it is dual port so it is possible to modify the CLUT at any time without visible glitches.
* The RGB DACs.
* The circuitry which shifts out pixels from the back line buffer to DACs.

# The Audio Subsystem

The recovered schematics documents it as "PANTHER DEVELOPMENT SYSTEM OTIS SOUND SYSTEM" and it is implemented as a daughter card with lot of glue logic in the SDKs found.

<table style="width:50%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img src="{{ site.url }}/images/atari-panther-2/otis.jpg"></td>
  </tr>
  <tr>
    <td>The sound daughterboard. OTIS in black. SRAM in red.</td>
  </tr>
</table>

This suggests that the audio subsystem was not yet finalized when the project was stopped and this was just the latest iteration.

The audio chip is not a custom ASIC. Instead, an off-the-shelf chip is used. It is an [Ensoniq ES5505  OTIS](https://github.com/vgmrips/vgmplay/blob/master/VGMPlay/chips/es5506.c) @ 8 Mhz. It has:
* 32 PCM channels
* Digital filters
* Stereo panning and volume

The PCM samples are stored in a dedicated 8 bit SRAM. It is implemented with a single 8x8K SRAM chip @ 8 Mhz (120ns).

This minimal amount of audio SRAM implies that samples have to streamed from the ROM. The Object Processor and its Copy Object can be used to implement this efficiently.

## The Developer Kit

<table style="width:50%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img src="{{ site.url }}/images/atari-panther-2/image_1.png"></td>
  </tr>
  <tr>
    <td>Internals of the developer kit (S. Walgenbach)</td>
  </tr>
</table>

The developer kit contains some additional components:
* A 2MB 16 bit ROM emulator based on 16 SRAM chips
* A MC6821 to read/write the ROM emulator via parallel port
* More glue logic

# Closing

In the next article we will explore how the Atari Panther renders graphics and we will review critically the advertised specs.
