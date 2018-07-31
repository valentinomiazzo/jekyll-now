---
layout: post
title: The Atari Panther - Part 1
---

The Atari Panther is a never released game console from the ‘90s. Its story is fascinating because of its connections. Its hardware is interesting for its strengths and shortcomings. Its fate is inspiring for what it could have been.

In this series of articles we’ll know more about the Atari Panther.

<table style="width:75%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img src="{{ site.url }}/images/atari-panther-1/image_0.png"></td>
  </tr>
  <tr>
    <td>A <a href="https://imgur.com/a/lbdFc">3D rendering</a> of the Atari Panther</td>
  </tr>
</table>

# The console

## Components

[Documents](http://www.atarimuseum.com/videogames/consoles/jaguar/Panther/index.htm) found in the recent years allow to well understand how the Atari Panther works and how it looks. The [schematics](https://www.chzsoft.de/asic-web/console.pdf) have been recovered by Christian Zietz. These, plus photos of the [developer kit](http://www.homecomputer.de/pages/panther.html) and the [developer manuals](http://www.atarimuseum.com/videogames/consoles/jaguar/Panther/PantherHW%20DocumentsFlare%20II.zip), allow to define its components.


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

lSRAM   = lower SRAM
uSRAM   = upper SRAM
Panther = Panther Chip
GS      = GameShifter Chip
CPU     = CPU
ROM     = Boot ROM
AC      = Audio Chip
EXP     = Expansion Port
CTG     = Cartridge Port</td>
```

Components architecture
* The CPU, a Motorola 68000 @ 16 Mhz
* The Panther chip (PC, 160 pin) @ 16 Mhz, it is the heart of the system and perform all these functions:
    * Support logic for the CPU
    * Bus arbiter for the main bus
    * Address decoding
    * Bridges the lower and upper data bus
    * Contains the Object Processor
    * Feeds the GameShifter chip via a 5 bit data bus and two 9 bit address bus. One with the start x and the other with the end x.
    * Generates clocks and video synchronization signals
* The [GameShifter](https://www.chzsoft.de/asic-web/console.pdf) chip (GS, 84 pin) @ 16 Mhz, contains:
    * A dual port CLUT
    * Two line buffers (a front and a back buffer)
    * The RGB DACs.
    * The circuitry which shifts out pixels from the back line buffer to DACs.
* The main RAM. 32 KB of SRAM @ 8 Mhz (120ns). It is composed of four 8KB chips on a 32 bit bus with each byte maskable
* The audio chip. An Ensoniq ES5505 OTIS @ 8 Mhz
* The audio RAM. 8 KB of SRAM @ 8 Mhz (120ns). It is composed of a single chip on a 8 bit bus.
* The BIOS ROM. 64 KB of ROM @ 4Mhz (250ns). It is composed of two 32KB chips on a 16 bit bus
* Discrete glue logic required by the audio subsystem and to read the 2 joystick ports
* A Cartridge port with 16 bit data bus and 24 bit address bus
* A Expansion port with 16 bit data bus and 24 bit address bus

<table style="width:50%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img src="images/atari-panther-1/image_1.png"></td>
  </tr>
  <tr>
    <td>Internals of the developer kit (S. Walgenbach)</td>
  </tr>
</table>

The developer kit contains some additional components:
* A 2MB 16 bit ROM emulator based on 16 SRAM chips
* A MC6821 to read/write the ROM emulator via parallel port
* More glue logic

## Specifications

These are the Panther specifications:
* The 68000 CPU is 16 bit, at 16 Mhz, can theoretically [reach](https://en.wikipedia.org/wiki/Instructions_per_second#MIPS) 2,8 MIPS .
    * Can read all the memory in the system
    * SRAM (4 mclk no wait states, a mclk is a 16 Mhz cycle, 62,5 ns)
    * Audio SRAM (4 mclk ?)
    * Boot ROM (4 mclk ?)
    * Cartridge ROM (6 mclk)
    * Cartridge RAM (6 mclk)
    * Panther registers (4 mclk)
    * GameShifter  (4 mclk)
        * registers
        * CLUT
        * line buffer (write only)
    * OTIS registers (4 mclk ?)
    * Joypad registers  (4 mclk ?)
* Up to 6MB of ROM on the cartridge @ 4 Mhz
* Up to 32KB of RAM on the cartridge @ 4 Mhz
* The screen resolution is 320x200 (NTSC)
* Two line buffers (a front and a back buffer) of 320 pixel with 5 bit depth.
    * The front buffer can be copied in a single clock cycle on the back buffer.
    * The Panther chip can only write the front line buffer @ 16 Mhz (62 ns).
    * The CPU can only write the front line buffer through the Panther chip @ 8 Mhz (125 ns, 2 mclk).
* The Color Look Up Table (CLUT)
    * has 32 colors with 18 bit depth (262K shades).
    * it is dual port so it is possible to modify the CLUT at any time without visible glitches.
* The graphic composition is done by the Panther Chip (PC) scan line per scan line in a double buffered line buffer executing an Object List (OL).
    * There is no limit to the Objects into the OL except the SRAM size and the time available to process it (a scan line, 64 usec).
    * There are multiple type of Objects, each with its function
        * *Bitmap* object
            * any size
            * 1,2,4,8 bit per pixel
            * zoomable and shrinkable
            * optionally Run Length Encoded
        * *Copy* an array from anywhere to the CLUT at a given scanline
        * *Branch* to another Object based on the scan line currently processed
        * *Copy* an array from anywhere to anywhere
        * *Write* a constant to a given address
        * *Add* a constant to a given address
        * *Interrupt* the 68000 at a given scan line
    * All the Objects, except the Bitmap Objects, can be stored also in ROM
    * Bitmap data can be stored in ROM or SRAM
* The [audio chip](https://github.com/vgmrips/vgmplay/blob/master/VGMPlay/chips/es5506.c) has
    * 32 PCM channels
    * Digital filters
    * Stereo panning and volume

# History

In 1989 Atari has a crowded lineup of aging 8 bit consoles (the 2600jr, the 7800, the XEGS) that are not able to contrast the very successful Nintendo NES in USA.

<table style="width:90%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/Atari-2600-Jr-FL.jpg"></td>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/300px-Atari-7800-Console-Set.jpg"></td>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/300px-Atari_XEGS.jpg"></td>
  </tr>
  <tr>
    <td>Atari 2600jr</td>
    <td>Atari 7800</td>
    <td>Atari XEGS</td>
  </tr>
</table>

<br/>

<table style="width:50%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/image_2.png"></td>
  </tr>
  <tr>
    <td>Nintendo NES</td>
  </tr>
</table>

In Europe the Commodore Amiga 500 is the king and great games are continuously [released](http://hol.abime.net/hol_search.php?&Y_released=1989). Its competitor, the Atari ST, has just a niche of the market.

<table style="width:90%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/300px-Amiga500_system.jpg"></td>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/300px-Atari_1040STf.jpg"></td>
  </tr>
  <tr>
    <td>Commodore Amiga 500</td>
    <td>Atari ST</td>
  </tr>
</table>

The 16 bit game console generation is rising from Japan with the NEC PC Engine released in 10/87 and the Sega Megadrive in 10/88. Atari is well aware of this.

Indeed, in 01/89 Sega contacts Atari asking to distribute the Megadrive in USA but Atari declines citing the high release price. This is the second time Atari rejects this kind of deals. It happened the same in ‘83 for the Nintendo NES.

The console hype is mounting also in Europe with Megadrive and PC Engine unofficially imported from Japan at crazy prices.

<table style="width:90%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/250px-PC-Engine-Console-Set.jpg"></td>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/250px-Sega-Mega-Drive-JP-Mk1-Console-Set.jpg"></td>
  </tr>
  <tr>
    <td>NEC PC Engine</td>
    <td>Sega Megadrive</td>
  </tr>
</table>

It is in Europe where another new console creates a lot of buzz on the specialized press. This is the [Konix Multisystem](https://en.m.wikipedia.org/wiki/Konix_Multisystem). It promises Amiga level graphics, better 3D and half the price thanks to custom chips containing a Blitter and a DSP. Nevertheless the Konix Multisystem never reaches the shelves because Konix goes out of cash in 03/89 before the release.

<table style="width:50%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/S667-02.jpg"></td>
  </tr>
  <tr>
    <td>Konix Multisystem</td>
  </tr>
</table>

In this scenario Atari decides that it must have a new console. A console based on the Atari STE is briefly considered but a better option is available.

The Konix Multisystem chips were designed in UK by a small unknown consulting company, Flare Technology. The founders are veterans that worked at Sinclair and at Amstrad.

Two of the members of Flare Technology, Martin Brennan and John Mathieson, are [contracted by Atari](http://www.konixmultisystem.co.uk/index.php?id=interviews&content=martin ) to work on two new consoles. Flare 2 is formed and the design of the Atari Panther and the Atari Jaguar begins in the Q1/89.

The two consoles are developed in parallel. The Jaguar is the real heir of the Konix chipset being a design based on framebuffer with blitter and RISC processors. The Panther is more about to productionize a concept already defined by Atari, a sort of 16/32 bit evolution of the Atari 7800.

In 08/89 the Sega Megadrive is released in USA as Sega Genesis and the NEC PC Engine as the NEC TurboGrafx-16.

In 09/89 Atari releases the Lynx portable console, a project acquired from the bankrupt Epyx and designed by two ex-Amiga engineers, Dave Needle and R.J. Mical.

<table style="width:50%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/300px-Atari-Lynx-I-Handheld.jpg"></td>
  </tr>
  <tr>
    <td>Atari Lynx</td>
  </tr>
</table>

The 1990 passes without big changes on the market, the NES and the Amiga 500 are still the leaders in USA and EU respectively.

The Megadrive is gaining ground but very slowly with around 4 or 5 titles [released every month](https://en.wikipedia.org/wiki/List_of_Sega_Genesis_games). In 09/90 the Megadrive is released in  EU.

In 11/90 the Nintendo Super Famicom is launched in Japan (Famicom is the Japanese name of the NES).

<table style="width:90%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/250px-Nintendo-Super-Famicom-Set-FL.jpg"></td>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/250px-SNES-Mod1-Console-Set.jpg"></td>
  </tr>
  <tr>
    <td>Nintendo Super Famicom</td>
    <td>Nintendo Super NES</td>
  </tr>
</table>

The system specifications are better than those of the Sega Megadrive, the launch titles are well received and clearly demonstrate the strong points of the hardware.

<table style="width:90%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/Supermarioworld.jpg"></td>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/SNES_F-Zero.png"></td>
  </tr>
  <tr>
    <td>Super Mario World</td>
    <td>F-zero</td>
  </tr>
</table>

Around the same time the Panther Development System ("Introduction to the Panther Development System" November 17, 1990) is ready.

The 1991 is a decisive year,  Sega is already in USA, Nintendo is coming and both are locking developers under exclusivity contracts.

At the start of 1991, after almost 2 years of work, some bugs are still present in the Panther chip (Some bugs related to the Palette Object are described in the "Introduction to the Panther Development System") and the audio subsystem is still very sketchy. Probably another spin of the Panther chip and a more integrated audio subsystem is needed. On the Jaguar side, the Blitter is almost complete and work is starting on the custom RISC processor (see [Jaguar’s netlist](http://atariage.com/forums/index.php?app=core&module=attach&section=attach&attach_id=197009)).

By 05/91 the Jaguar has the bulk of the work done and a refinement, enhancement and bug fixing phase is starting (see Jaguar’s netlist).

With such time pressure, and the Atari Jaguar development proceeding faster than expected, Atari decides to kill the Panther project because it would result in a launch too near to the launch of the Jaguar.

<table style="width:50%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/image_3.png"></td>
  </tr>
  <tr>
    <td>The project cancellation memo 05/91 (Atari museum)</td>
  </tr>
</table>

In 08/91 the Nintendo SNES is released in USA. Sega cuts the price of the Megadrive and releases the very successful "Sonic the Hedgehog". The 16 bit war starts between Sega and Nintendo.

<table style="width:50%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/image_4.png"></td>
  </tr>
  <tr>
    <td>Sonic the Hedgehog</td>
  </tr>
</table>

It is only in late ‘92, with the release of the SNES in EU and the release of Super Street Fighter 2 for SNES, that the market is completely mature and everyone wants one of these 16 bit consoles.

Unfortunately the Jaguar is still in development after more than 3 years of work and more than 1  year since the Panther project was stopped.

<table style="width:50%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/image_5.png"></td>
  </tr>
  <tr>
    <td>Super Street Fighter 2</td>
  </tr>
</table>

Eventually, the cartridge based Atari Jaguar is released in USA in 11/93 for $250, one year before its competitors: the CD based, Sega Saturn and the Sony Playstation.

The development lasted more than 4 years and around 30 months passed from closing the Panther project to releasing the Jaguar. By contrast, the Sega Saturn was developed in 2 years and the Sony PlayStation in just 1 year.

<table style="width:50%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/image_6.png"></td>
  </tr>
  <tr>
    <td>Atari Jaguar</td>
  </tr>
</table>

<table style="width:90%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/320px-Sega-Saturn-Console-Set-Mk1.jpg"></td>
    <td><img style="vertical-align:middle;" src="images/atari-panther-1/320px-PSX-Console-wController.jpg"></td>
  </tr>
  <tr>
    <td>Sega Saturn</td>
    <td>Sony Playstation</td>
  </tr>
</table>

Being the first on the market doesn’t help the Jaguar. Sony’s huge investment buys the market and the developers pushing Atari and Sega out of the market.

# Timeline

* 02/86 Nintendo NES released in US for $90

* 10/87 NEC PC Engine is released in Japan

* 07/88 World wide RAM shortage begins

* 10/88 Sega Megadrive is released in Japan

* 12/88 NEC PC Engine CD-ROM is released in Japan

* 01/89 Sega proposes to Atari to sell the Megadrive in US

* 03/89 Konix runs out of cash

* Q1/89 Flare 2 is created. Panther and Jaguar design starts

* 04/89 SNK Neo Geo is released in Japan

* 06/89 World wide RAM shortage ends

* 08/89 Sega Genesis is released in US for $200

* 08/89 NEC TurboGrapx-16 is released in US

* 09/89 Atari Lynx is released in US

* 09/90 Sega Megadrive is released in EU

* 11/90 Nintendo Super NES is released in Japan

* 12/90 Nintendo NES is the leader in US and Japan, Commodore Amiga 500 is the leader in Europe

* 05/91 Atari kills the Panther project

* 06/91 Sega Megadrive price drops to $150

* 08/91 Nintendo SNES is released in US for $200

* 12/91 Sega Mega CD is released in Japan

* 06/92 Nintendo SNES is released in EU

* 10/93 3DO is released in US

* 11/93 Atari Jaguar is released in US for $250

* 11/94 Sega Saturn is released in US for $400

* 12/94 Sony Playstation is released in US for $300

* 09/95 Atari Jaguar CD is released
