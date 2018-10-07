---
layout: post
title: The Atari Panther - Part 2
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

# Recap

... History ...

# The console

[Documents](http://www.atarimuseum.com/videogames/consoles/jaguar/Panther/index.htm) found in the recent years, [schematics](https://www.chzsoft.de/asic-web/console.pdf) recovered by Christian Zietz, photos of the [developer kit](http://www.homecomputer.de/pages/panther.html) and the [developer manuals](http://www.atarimuseum.com/videogames/consoles/jaguar/Panther/PantherHW%20DocumentsFlare%20II.zip)  allow to well understand how the Atari Panther works and how it looks.

## Architecture

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

The machine has a unified memory architecture with some twists.
All the components are connected trough an address bus and a 16 bit data bus (a,ud) to a 32 bit Static RAM block.
The 32 bit SRAM is composed of two 16 bit SRAM banks (lSRAM and uSRAM).
The Panther Chip works as a de-multiplexer connecting the right SRAM bank based on the selected address.

Odd words are connected to the lower bank through the Panther Chip and another 16 bit data bus (ld).
```
                             a
...─┬──...───────────┬─────────────┐
    │                │      ld     │
  [CPU]          [Panther]──────[lSRAM]
    │                │             :
...─┴──...───────────┴...
```

Even words are connected to the upper bank directly.

```
                             a
...─┬───...──────────┬─────────────┐
    │                │             │
  [CPU]          [Panther]...      |
    │                │      ud     │
...─┴───...──────────┴──────────[uSRAM]
```

The Panther Chip is connected to both banks so it is the only component capable of 32 bit data accesses.

We can say that the Panther is a hybrid 16/32 bit machine.

## RAM and ROM

The main SRAM amounts to just 32 KB. It is organized in two 16 bit banks. Each bank is composed of two chips with 8K x 8 bit.

Since each chips can be individually enabled it is possible to read/write just some of the bytes during an access.
This permits byte accesses but it also permits to mask some bytes during a 32 bit write access.

For example if the destination long word contains 0xAABBCCDD, the data bus contains 0x00112233 and the 3rd chip is not selected then the 3rd byte is masked.

    0xAABBCCDD : SRAM content
    0x00112233 : write data
      eeee--ee : enable mask
    0x0011CC33 : new SRAM content

As far I know this feature is not used on the Panther, anyway keep it in mind for when we'll talk about how the Panther could have been enhanced.

The main SRAM run @ 8 Mhz (125ns, 2 mclk).
<br>*NOTE*: a mclk is a 16 Mhz cycle, 62,5 ns.

The Panther contains also 64KB of 16 bit bootstrap ROM. It is composed of two 32K x 8 bit chips running @ 8 Mhz (125ns, 2 mclk).

## Cartridge and Expansion ports

The Cartridge port has a 16 bit data bus (ud) and a 24 bit address bus (a).
It supports:
* Up to 6MB of ROM @ 4 Mhz (250ns, 4 mclk)
* Up to 32KB of RAM @ 4 Mhz (250ns, 4 mclk)

This means a bandwidth of 8 MB/s.

The Expansion port has a 16 bit data bus (ud) and a 24 bit address bus (a). Is not clear what is the access speed.

## CPU

The CPU is a Motorola 68000 @ 16 Mhz. It can theoretically [reach](https://en.wikipedia.org/wiki/Instructions_per_second#MIPS) 2,8 MIPS.

Thanks to the shared bus, the CPU can access all the other components in the system. This is different from other consoles where different buses exists and the CPU cannot access them directly.

The 68000 requires 2 mclk to prepare the access and then it waits 2 mclk for the access. If the accessed memory requires more than 2 mclk then the 68000 has to be slowed down requesting wait states. For example, since the Cartridge ROM responds in 4 mclk then 2 wait states are introduced.

All the components in the system have a 2 mclk access cycle except the Cartridge RAM and ROM which need 4 mclk. This means that the 68000 has no wait states except when accessing the Cartridge RAM/ROM.

Therefore, when the CPU is executing code from Cartridge ROM it requires 6 mclk to fetch 1 instruction instead of 4 mclk. It means that is 50% slower than when executing code from main SRAM.

## The Panther Chip

The Panther Chip (PC) has 160 pins and works @ 16 Mhz. It is the "heart" of the system and perform all these functions:
* Support logic for the CPU
* Bus arbiter for the main bus
* Address decoding
* Bridge for the lower data bus
* Generator of clocks and video synchronization signals
* Embed the Object Processor
* Feed the GameShifter chip via a 5 bit data bus and two 9 bit address bus.

The Object Processor is where the graphic is generated. To understand how it works we can draw some similarities with how modern 3D graphics card works.

In a 3D card, every frame, a GPU executes a list of commands (like draw a triangle, change texture, etc...) and draws in a dedicated high speed frame buffer.

The Object Processor is similar. It draws rectangles instead of triangles, the buffer is just one scanline and the list of commands (Object List, OL) is execute at every scanline instead of every frame. The line buffer is actually on the GameShifter Chip.

There is no limit to the Objects into the OL except the SRAM size and the time available to process it (a scan line, 64 usec). In pseudo-code it looks like:

``` C
int y=0
do {
  if(isScanLineVisible(y)) {
    Object* o = objectList.head;
    while(o!=null && !isScanLineTimeComplete()) {
      switch(o.type) {
        case BITMAP:
          int l = o.y - y;
          if(l>=0 && l<o.h) {
            drawLineInBufferAtX(o.bitmapLines[l], lineBuffer, o.x);
          }
          break;
        case CLUT:
          ...      
      }  
      o = o.next;    
    }
    waitScanLineTimeComplete();
    copyAndClear(lineBuffer,backLineBuffer);
  }
  if(isLastScanLine(y)) { y=0; } else { y=y+1; }
} while(true)
```

This is quite similar to how the Atari 7800 works and very different from framebuffer systems (Amiga) and sprites & tiles systems (Super NES and Megadrive).

There are multiple type of Objects, each with its function:
* *Bitmap* object
    * any size
    * 1,2,4,8 bit per pixel
    * zoomable and shrinkable
    * optionally Run Length Encoded
* *CLUT* object copies an array of byte from anywhere to the CLUT at a given scanline
* *Branch* to another Object based on the scan line currently processed
* *Copy* an array from anywhere to anywhere
* *Write* a constant to a given address
* *Add* a constant to a given address
* *Interrupt* the 68000 at a given scan line

All the Objects, except the Bitmap Objects, can be stored also in ROM. Bitmap data can be stored in ROM or SRAM.

Being able to fetch graphic directly from the ROM is another important difference compared to Super NES and Megadrive. These machines can fetch graphic only from dedicated graphic RAM and therefore are continuously streaming data from ROM to the graphic RAM. In this regard the Panther resembles the SNK Neo Geo.

On the Developer Kit the Panther Chip is implemented on a [Toshiba TC110G32](https://datasheetspdf.com/mobile-pdf/905878/Toshiba/TC110G32.html). This is a Gate Array not a custom ASIC. It is not clear if actual ASICs were planned or ever produced. The chip contains 32K gates (13K usable) with a delay of 0.6 ns. By contrast the Amiga Agnus chip is composed of about [7K gates](https://retrocomputing.stackexchange.com/questions/3000/how-many-gates-does-each-chip-in-ocs-have?rq=1). With a cycle of 62.5 ns (16 Mhz) it should mean that pipeline stages are max 10 gates deep.

## The GameShifter Chip

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

## The Audio Subsystem

The audio chip is not a custom ASIC. Instead, an off-the-shelf chip is used. It is an [Ensoniq ES5505  OTIS](https://github.com/vgmrips/vgmplay/blob/master/VGMPlay/chips/es5506.c) @ 8 Mhz. It has:
* 32 PCM channels
* Digital filters
* Stereo panning and volume

In the Development Kit, the audio subsystem contains lot of glue logic and looks very sketchy. The PCM samples are stored in a dedicated 8 bit SRAM. It is implemented with a single 8x8K SRAM chip @ 8 Mhz (120ns).

This minimal amount of audio SRAM implies that samples have to streamed from the ROM. The Object Processor and its Copy Object can be used to implement this efficiently.

The ES5505 is not just [sample player](https://www.youtube.com/watch?v=nYbEl51L6CQ#t=4m30s), it can be used as a powerful [synthesizer](https://www.youtube.com/watch?v=cteo1xDEzgo). Indeed this chip is inside the famous [Ensoniq EPS 16 plus](http://www.vintagesynth.com/ensoniq/ens_eps16.php), many [arcade machines](https://www.youtube.com/watch?v=CCiLF5pKaS8&list=PLznWdSFQxsPshMJiGluFAAp2f-T-r0l4a) of the time and PC cards.

As you can hear, the Panther is fare beyond both the Megadrive and the SNES on the audio side.

## The Developer Kit

<table style="width:50%;font-size:65%;margin:auto;text-align:center;">
  <tr>
    <td><img src="{{ site.url }}/images/atari-panther-1/image_1.png"></td>
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

We know now the story of the Atari Panther and what it is inside. In the next article we will look at the competition and we will start to dig in how it works and how powerful it is.
