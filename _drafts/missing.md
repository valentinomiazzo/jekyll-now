---
layout: post
title: The Atari Panther - Part 2
---
# Going deeper


TODO: move this is a chapter dedicated to how the graphic works ...
``` C
int y=0   # current scanline
do {
  if(isScanLineVisible(y)) {
    void* op = objectListPointer;
    while(op!=null && !isScanLineTimeComplete()) {
      Object obj = fetchObject(op)
      switch(obj.type) {
        case BITMAP_OBJECT:
          int l = obj.y - y;   // current bitmap line
          if(l>=0 && l<obj.h) {
            drawLineInBufferAtX(o.bitmapLines[l], lineBuffer, obj.x);
          }
          break;
        case CLUT_OBJECT:
          ...      
          ...      
      }  
      op = obj.next;    
    }
    waitScanLineTimeComplete();
    copyAndClear(lineBuffer,backLineBuffer);
  }
  if(isLastScanLine(y)) { y=0; } else { y=y+1; }
} while(true)
```

This is quite similar to how the Atari 7800 works and very different from framebuffer systems (Amiga) and sprites & tiles systems (Super NES and Megadrive).

Note how Bitmap Objects that appear early in the Object List are shown under the others.

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

All the Objects, except the Bitmap Objects, can be stored also in ROM. This because Bitmap Objects are updated during the drawing. A Bitmap Object contains a pointer to the bitmap data. This pointer can point to ROM or SRAM.



The ES5505 is not just [sample player](https://www.youtube.com/watch?v=nYbEl51L6CQ#t=4m30s), it can be used as a powerful [synthesizer](https://www.youtube.com/watch?v=cteo1xDEzgo). Indeed this chip is inside the famous [Ensoniq EPS 16 plus](http://www.vintagesynth.com/ensoniq/ens_eps16.php), many [arcade machines](https://www.youtube.com/watch?v=CCiLF5pKaS8&list=PLznWdSFQxsPshMJiGluFAAp2f-T-r0l4a) of the time and PC cards.

As you can hear, the Panther is fare beyond both the Megadrive and the SNES on the audio side.


## Declared performance

While the Panther was in development, Atari started a press campaign and communicated wonderful specifications. As usual in these cases these figures don’t quite represent the truth, let’s see why.

![image alt text](image_7.png)

Specifications declared to the press (from [Internet Archive](https://web.archive.org/web/20031207084119/http://www.atari-explorer.com:80/Panther-Spec.htm))

The specs don’t need comments except for:

* "7860 colors/screen"
This is misleading and comes from the idea of changing the 32 entries of the CLUT at every visible scanline. In NTSC, 240*32=7860. While doable, it can be used only for static images.

* "32 Mhz display processor"
This is misleading, the Panther has a 32 Mhz crystal but the master clock is 16 Mhz and some parts work at 8 Mhz or even 4 Mhz.

* "2000 simultaneous sprites"
This is misleading and comes from the footprint of a Bitmap Object which is 16 bytes long. By filling all the SRAM with Bitmap Object definitions we can reach 32*1024/16 = 2048 Bitmap Objects per frame. It is doable but the CPU is locked out of the main bus most of the time and has little time to update the Object List.

* "Fast hardware addition for object manipulation"
This is not so relevant and refers to the ability of increment a memory location through the Object Processor.

## CPU speed

The Motorola 68000 has an instruction cycle composed of 4 clock cycles. During the first 2 clock cycles the address and data bus are not used but driven.

![image alt text](image_8.png)

Motorola 68000 (NOTE:  CPU Grave Yard)

In systems like the Commodore Amiga or the Atari ST this is leveraged to time multiplex the CPU and the graphic subsystem on the same RAM.

The RAM must have a read/write cycle of 2 clock cycles. Additional chips like the 74LS373, 74LS244 and 74LS245 are used to disconnect the 68000 from the bus (address, data, control) during the first 2 clock cycles so other chips can control the bus.

The access pattern is:

GGccGGcc....

G = Graphig subsystem

c = CPU

This is very smart because back then RAM was faster than CPUs and this allowed to fully utilize the RAM bandwidth.

For example, the Atari ST has a CPU clock of 8 Mhz and DRAM chips with read/write cycle of 250 ns (4 Mhz). With 4 Mhz and 16 bit the DRAM bandwidth is 4*2 = 8 MB/s. Half of the bandwidth goes to the 68000, half to the graphic subsystem. The 68000 @ 8 Mhz can execute 8/4 = 2 MIPS but since some instructions require more than 1 instruction cycle, 1.4 MIPS is a more correct estimation (NOTE:  https://en.wikipedia.org/wiki/Instructions_per_second#MIPS ).

The Panther has a 68000 @ 16 Mhz and the SRAM has a read/write cycle of 125 ns (8 Mhz) so the 68000 can theoretically run at full speed and reach 2.8 MIPS.

Anyway the schematics clearly show that the 68000 is directly connected to the SRAM. There is not decoupling logic (74LS244, 74LS373 & similar) to detach it from the main bus during the idle clock cycles. This means that CPU and Panther Chip cannot share the bus with the fine time interleaving described above.

<table>
  <tr>
    <td>                         
</td>
  </tr>
</table>


68000, SRAM, Panther connections

The developer manual confirms this: the Panther Chip acquires the main bus at the beginning of every scanline and releases it only when it completes the processing of the Object List. When this happens the CPU gets the main bus.

Indeed, the Panther is connected to the BR, BG, BGACK pins (NOTE:  read here for more details) of the 68000. This confirms that the Panther chip can steal the bus from the 68000.

The access pattern is:

vblank  scanlines: __cc__cc....__cc__cc....__cc

visible scanlines: PPPPPPPP....PPPP__cc....__cc

                               ^--end of obj list processing

P = Panther chip

c = CPU

_ = Not used

This also means that in the worst case the CPU is always locked out of the main bus. A less severe, but still problematic scenario, is when there are so many objects on the screen that the CPU can work only during the Vblank.

In NTSC this means (262-200)/262 = 24% of time.

In PAL (312-256)/312 = 18% of time.

Plus, if the CPU is always reading from ROM then an instruction cycle takes 6 clocks instead of 4. The Panther can induce wait states via the DTACK pin.

NTSC 24% * 4/6 = 16% → 0.45 MIPS.

PAL 18% * 4/6 = 12% → 0.33 MIPS.

So if pushed too much on the graphic side, the Panther’s 68000 @ 16 Mhz (0.45 MIPS) can be substantially slower than the 68000 @ 8 Mhz of an Atari ST or a Sega MegaDrive (around 1.4 MIPS).

It is also clear that half of the SRAM bandwidth is wasted when the CPU is working.

The Sega Megadrive has 64KB of SRAM dedicated to the 68000 @ 7,6 Mhz (around 1.4 MIPS) where it can work at full speed. When it is time to manipulate graphic data it has to access the VRAM, which is not mapped on its address space, via IO ports with wait states.

The Nintendo SNES has a 65816 CPU @ 3,58 Mhz (1.7 MIPS, if efficient as the 6502). While it crunches more MIPS than the Megadrive it is actually slower because the data bus is 8 bit wide instead of 16. It also has just three 16 bit registers instead of sixteen 32 bit registers. Like on the Megadrive, VRAM is not mapped on the CPU address space and has to be accessed via IO ports with wait states.

When compared to SNES and Megadrive, the Panther has a CPU which is faster and an architecture more adapt to pixel manipulation, IF the CPU is not bottlenecked by the Panther Chip.

## Sprites and buffers

Early systems like the Atari 2600, Atari XL/XE, Commodore C64 or even the Commodore Amiga share a similar sprite subsystem, something we can call "fetch and trigger".

At the beginning of every scanline the graphic data of N sprites is fetched from memory, if they are visible at that scanline. The data is stored in N temporary registers. When the x position of a given sprite is reached by the raster beam this starts the shifting out of the data in the temporary register toward a mixer that mixes it with the data coming from other sprites and the playfield. Usually N is equal or less than 8.

![image alt text](image_9.png)

"Fetch and trigger" circuitry on the Commodore Amiga (NOTE:  Source: Amiga Hardware Reference Manual)

Later systems like the Atari 7800, Atari Panther, Nintendo SNES, Sega Megadrive, SNK Neo Geo use a more scalable approach based on line buffers.

Some fast RAM contains a list of sprites (Objects in the Panther). A video processor, at every scanline, processes the whole list from the beginning. If a sprite of the list intersects the current scanline then its graphic data is fetched from the graphic memory (e.g. from ROM) and copied in to the line buffer.

Sprite after sprite the graphic data accumulates on the line buffer. When the scanline time (64 usec) is elapsed, the line buffer is copied on a line back buffer. The line back buffer is displayed on the screen while the line buffer is cleared and filled again.

While other consoles of the time have both sprites and layers, the Panther has only sprites but this is offset by the fact that they could have any size. Another console with just sprites is the SNK Neo Geo. Another difference compared to Megadrive, SNES and Neo Geo is the absence of the concept of tiles. This means that tiles must be simulated by using Bitmap Objects. While possible this is quite inefficient on the Panther.

The Panther can fetch bitmap data directly from the cartridge ROM. This sets it aparts from the Sega Megadrive and the Nintendo SNES where bitmap graphic must be first copied in VRAM. On these machines, the best games are constantly streaming data from ROM to VRAM to increase the perceived graphic variety. On Panther this is not necessary and this simplifies the development and ensures more variety.

One could argue that the Panther doesn’t have a frame buffer like the Commodore Amiga or the Atari ST. This is partially true. Indeed it is possible to have a Bitmap Object large like the screen and that fetches bitmap data from the SRAM. This is a frame buffer and even a linear (NOTE:  A linear frame buffer is one where the address of a pixel can be computed with something simple like a(x,y)=y*c+x*d) one.

What makes it impractical is the limited amount of SRAM on the console. 320x200 pixels in 16 colors require 320*200/2 = 32000 bytes. This leaves 768 bytes for the rest and usually you need two frame buffers for glitch free animations.

Given this limitation, the Panther is very different from home computers like the Commodore Amiga or the Atari ST where the graphic composition was done primarily on frame buffers. The graphic composition is done primarily via Bitmap Objects (Sprites).

Other consoles like the Sega Megadrive and the Nintendo SNES can setup a frame buffer too but it is harder to access by the CPU because it is in VRAM out of the CPU address space and not linear. The Neo Geo can only fetch graphics from ROM so cannot have a frame buffer without some special cartridge.

## Colors and resolution

The Panther has a Color Look Up Table of 32 entries of 18 bit each (262K shades).

The Sega Megadrive has a CLUT with 64 entries of 9 bits each (512 shades), the Nintendo SNES has a CLUT of 256 entries of 15 bits each (32K shades).

262K shades are more than any console of the era, but 32 entries are very few for a machine of that period. Hopefully, both the CPU and the Panther Chip (via Palette Object) can manipulate the CLUT. Even better the Panther Chip can change the CLUT automatically at a given scanline (see Palette Object).

We can imagine that Panther games would have used Palette objects quite extensively to appear as colorful as the competitors in other game consoles.

The Panther as a fixed resolution of 320*200 pixels (NTSC). The SNES has 256*224 or 512*224, the Megadrive has 256*224 or 320*224.

Arcade games of the era often has an horizontal resolution of 320 or 256 pixels. When a game is ported from an Arcade machine to a game console, it is necessary to convert the graphic. When the graphic resolution doesn't match (for example the Arcade has 256 pixel and the console has 320 pixel) the graphic has to be resampled. This often requires lot of manual refinement. In these context, the Megadrive simplifies the process by supporting both 320 and 256 horizontal pixels.

An interesting feature of the Panther is that the Bitmap Object descriptor contains a CLUT offset. This is added to the value encoded in the pixel data to define the value written in the line buffer. For example if a pixel has a value of 3 and the CLUT offset is 7 then 10 is written.

For contrast, the Megadrive has a CLUT with 64 entries splitted in 4 banks of 16. Every sprite or tile has to select one of the 4 banks. If two sprites has 8 different colors and 8 common colors they must use two different banks wasting 8 CLUT entries. Conversely, on the Panther the two colors set can overlap over the common colors.

Megadrive case:

FEDCBA98**76543210** ← Sprite 1

PONMLIGH**76543210** ← Sprite 2

Panther case:

┌───────────┐Sprite 1

FEDCBA98**76543210**PONMLIGH

        └───────────┘Sprite 2

It can be said that, while the CLUT on the Panther is smaller than the one in the Megadrive, it is more usable and therefore the two machines are equivalent. This is not true for the SNES, its 256 entries CLUT makes it more colorful than the Panther.

## Sprites per line

It is interesting to study how many sprites the Panther can push and compare it with the competitors.

The Megadrive and SNES have less flexible but more specialized sprite hardware and precise constraints. Megadrive has max 80 sprite per frame, 20 sprites per scanline, 320 sprite pixel per scanline. The SNES has max 128 sprites per frame, 32 sprites per scanline, 272 sprite pixel per scanline.

The Panther has not a hard limit on the sprites per frame or per scanline. The limits come from the amount of SRAM where the object list is stored and the finite amount of time in a scanline.

It has a ROM bandwidth of 8 MB/s. In other terms, there are 256 ROM accesses per scanline. With 4 bit per pixel (to be on pair with competition) this means 4 pixel per ROM access and therefore a theoretical limit of 1024 sprite pixel per scanline. This is also the limit of the pixels that can be written on the line buffer during a scanline (1 pixel/mclk).

This looks awesome when compared to the 320 and 272 of the competitors but the comparison is unfair because on the Panther the sprites have to be used also for the background layers.

The Megadrive has 2 layers of 4 bit per pixel. Therefore it draws 320*2 + 320 = 960 pixels per scanline. The Panther would need 320*2/4 + 320/4 = 240 ROM accesses and 960 frame buffer accesses. It can do it.

The SNES has many video modes but the most comparable is mode 1 with two layer in 4 bit per pixel and one layer in 2 bit per pixel. Therefore it draws 256*3 + 272 = 1040 pixels per scanline. The Panther would need 256*2/4 + 256/8 + 262/4 = 228 ROM accesses and 1040 frame buffer accesses. Almost there if just 256 pixel per scanline are used. Far behind if 320 pixels have to be used to cover the entire screen.

From a pure bandwidth point of view, the Panther is in good shape but this is not the full story. Panther's weak and strong point is the unified memory architecture.

The Panther Chip has to fetch the Object List from the SRAM eating bandwidth. If many objects have to be parsed to compose the scanline then few objects can be draw in time. To see how severe this can be, let's try to simulate a layer made if tiles of 8*8 pixels.

The Panther doesn't have the concept of layers or tiles, therefore, every tile has to be an Object. The Panther Chip needs 4 reads and 2 writes on SRAM for every Bitmap Object. Each SRAM access requires 2 mclks. There are 40 tiles and 1024 mclks per scanline (16 MHz). At 4 bit per pixel, 8 pixels require 2 ROM accesses. A ROM access requires 4 mclks. In total we have: 40*6*2 + 40*2*4 = 800 mclks. So just one tiled layer almost exhausts the scanline, and it is conservative since we didn’t consider the needed Branch Objects.

Also the Object List size is remarkable, we have 40*25 = 1000 tiles, every tile requires a Bitmap Object with a footprint of 4 long words. This amounts to: 40*25*4*4 = 16000 bytes, almost half the available SRAM. Also in this case, it is conservative since we didn’t consider the needed Branch Objects.

SNES and Megadrive have RAM and bus dedicated to the fetching of sprites definitions and therefore there are hard limits and no tradeoffs to make. Layers are always tiled.

One can argue that a tile layer such defined is extremely more flexible but the reality is that this makes porting games from Arcade machines or competitor consoles complex.

Overall, this shows the cross and delight of the Panther, Object Lists and the unified SRAM make is very flexible but also waste lot of resources when you have to port games designed for other more common hardware.

## Zooming sprites

This is a feature of the Panther that makes it unique among the 16 bit consoles.

Bitmap Objects can be zoomed or shrunk. The X and Y scale can be specified independently.

How this impact the drawing performance? We can answer this question by comparing a Bitmap Object zoomed or shrunk to size W,H with one not zoomed of size W,H.

In case of a zoomed object we have to fetch less bitmap data. For example if zoom is R>1 in X direction and K>1 in Y direction then the object requires 1/R th the data. This is because W/R pixels are repeated to fit W pixels and the Panther chip keeps a buffer of the last fetched pixels. It is not 1/(R*K) because the Panther works per scanline and cannot buffer an entire line of bitmap data, therefore every line even if identical to the previous has to be processed again.

<table>
  <tr>
    <td>1</td>
    <td>1</td>
    <td>2</td>
    <td>2</td>
    <td>3</td>
    <td>3</td>
    <td>4</td>
    <td>4</td>
  </tr>
  <tr>
    <td>1</td>
    <td>1</td>
    <td>2</td>
    <td>2</td>
    <td>3</td>
    <td>3</td>
    <td>4</td>
    <td>4</td>
  </tr>
  <tr>
    <td>5</td>
    <td>5</td>
    <td>6</td>
    <td>6</td>
    <td>7</td>
    <td>7</td>
    <td>8</td>
    <td>8</td>
  </tr>
  <tr>
    <td>5</td>
    <td>5</td>
    <td>6</td>
    <td>6</td>
    <td>7</td>
    <td>7</td>
    <td>8</td>
    <td>8</td>
  </tr>
  <tr>
    <td>9</td>
    <td>9</td>
    <td>10</td>
    <td>10</td>
    <td>11</td>
    <td>11</td>
    <td>12</td>
    <td>12</td>
  </tr>
  <tr>
    <td>9</td>
    <td>9</td>
    <td>10</td>
    <td>10</td>
    <td>11</td>
    <td>11</td>
    <td>12</td>
    <td>12</td>
  </tr>
  <tr>
    <td>13</td>
    <td>13</td>
    <td>14</td>
    <td>14</td>
    <td>15</td>
    <td>15</td>
    <td>16</td>
    <td>16</td>
  </tr>
  <tr>
    <td>13</td>
    <td>13</td>
    <td>14</td>
    <td>14</td>
    <td>15</td>
    <td>15</td>
    <td>16</td>
    <td>16</td>
  </tr>
</table>


Example of a 4x4 object with 4 bpp and R=K=2.
In bold the pixel that triggers the data fetch (8).

<table>
  <tr>
    <td>1</td>
    <td>2</td>
    <td>3</td>
    <td>4</td>
    <td>5</td>
    <td>6</td>
    <td>7</td>
    <td>8</td>
  </tr>
  <tr>
    <td>9</td>
    <td>10</td>
    <td>11</td>
    <td>12</td>
    <td>13</td>
    <td>14</td>
    <td>15</td>
    <td>16</td>
  </tr>
  <tr>
    <td>17</td>
    <td>18</td>
    <td>19</td>
    <td>20</td>
    <td>21</td>
    <td>22</td>
    <td>23</td>
    <td>24</td>
  </tr>
  <tr>
    <td>25</td>
    <td>26</td>
    <td>27</td>
    <td>28</td>
    <td>29</td>
    <td>30</td>
    <td>31</td>
    <td>32</td>
  </tr>
  <tr>
    <td>33</td>
    <td>34</td>
    <td>35</td>
    <td>36</td>
    <td>37</td>
    <td>38</td>
    <td>39</td>
    <td>40</td>
  </tr>
  <tr>
    <td>41</td>
    <td>42</td>
    <td>43</td>
    <td>44</td>
    <td>45</td>
    <td>46</td>
    <td>47</td>
    <td>48</td>
  </tr>
  <tr>
    <td>49</td>
    <td>50</td>
    <td>51</td>
    <td>52</td>
    <td>53</td>
    <td>54</td>
    <td>55</td>
    <td>56</td>
  </tr>
  <tr>
    <td>57</td>
    <td>58</td>
    <td>59</td>
    <td>60</td>
    <td>61</td>
    <td>62</td>
    <td>63</td>
    <td>64</td>
  </tr>
</table>


Example of a 8x8 object with 4 bpp no zoom.
In bold the pixel that triggers the data fetch (16).

This 1/R reduction of the fetched data results in a gain of speed only if the Bitmap Object uses 8 bit per pixel. With 8 bit per pixel the ROM bandwidth (256 access/scanline * 2 pixel/access = 512 pixel/scanline) is lower than the line buffer bandwidth (1024 access/scanline = 1024 pixel/scanline) and becomes the bottleneck. With zoom the ROM bandwidth sustains 512*R pixel/scanline and can therefore draw more pixels. For R > 2 the line buffer bandwidth is saturated and no further drawing speed can be gained.

With less than 8 bit per pixel the line buffer bandwidth is always the bottleneck and saved ROM accesses don’t produce any speed up because nothing can use the ROM or the main bus while the Panther Chip is owning it.

It is fair to say that 8 bit per pixel Bitmap Objects are rarely used since they use more bandwidth, ROM space and the CLUT has 32 bit entries (5 bits). So, typically zoom is free (no gains, no penalties).

In case of a shrunk object we have to fetch more data. For example, if shrink is r>1 in X direction and k>1 in Y direction then the object requires r times the data. It is not r*k because Y shrink just skip the invisible lines so nothing has to be fetched. So, the higher is r the lower is the drawing speed.

Hopefully, when r becomes >= 16/BitPerPixel the drawing speed doesn’t decrease anymore. For example with 4 bit per pixel to draw an object with r=4 costs like to draw one with r=32. The reason for this behaviour is that when you have to do one ROM access for every pixel then the situation cannot go worse than this.

![image alt text](image_10.jpg)

mclk/pixel VS r for a 4 bpp Bitmap Object

The SNES as something slightly similar to zoom. The video Mode 7 allows to apply an affine transformation to one layer. This permits to zoom and rotate such layer. If the parameters are changes dynamically along the screen this permits to add perspective. For example, F-Zero and Mario Kart use this technique.

<table>
  <tr>
    <td>
F-Zero</td>
    <td>
Mario Kart</td>
  </tr>
</table>


Something like this can be emulated on the Panther by using the CPU to render it in SRAM. Since two frame buffers are needed to avoid glitches they must cover less than half the screen to fit in SRAM and leave some space for the rest.

But are games like Sega Afterburner or Sega Outrun where the Panther runs circles around the Megadrive and the SNES.

<table>
  <tr>
    <td>
Afterburner (arcade)</td>
    <td>
Outrun (arcade)</td>
  </tr>
</table>


As we have seen when discussing sprites, the Panther can draw 1024 pixel/scanline. This is 3,2 times the screen.

On a game like Afterburner on the horizon there are about 16 sprites covering all the screen width (320 pixel). At the bottom of the screen there are about 6 sprites covering all the screen width. The game can tilt the horizon but let’s assume it is aligned with the scanlines for now. Every row of sprites is partially covered by another row below and it partially covers another row of sprites above. Let’s assume the row below and the row above don’t overlap.

|

||  <--- row above

 ||  <--- row

  ||  <--- row below

   ||

    |

In this case every scanline is covered by max 2 rows of sprites. We know that the Panther has a penalty for every parsed Bitmap Object. Therefore the worst case is near the horizon where there are scanlines with 16*2 sprites of about 320/16=20 pixel each. There we have (assuming 4bpp) 16*2*(2*6+4*20/4) = 1024 mclk exactly the draw limit. This means we can draw the ground but not the airplanes. At the bottom of the screen we have 6*2 sprites of 320/6=~56 pixel each resulting in 6*2*(2*6+4*56/4) = 816 mclk. Around 200 mclk remain available for the airplanes.

If the Object List is ordered from near to far then the hardware will automatically avoid to draw the sprites behind other sprites when there is too much stuff on a scanline. This is not ideal because results in flickering but at least is less noticeable.

A compromise can be obtained by using a less dense setup. Instead of 16 sprites we can use 12 sprites of 20 pixels each and offset horizontally the rows so they are not aligned horizontally.

┌─────┐──┌─────┐──┌─────┐──┌─────┐──┌─────┐──┌─────┐

│─────│──│─────│──│─────│──│─────│──│─────│──│─────│

│───┌─────┐──┌─────┐──┌─────┐──┌─────┐──┌─────┐────│

└───│─────│──│─────│──│─────│──│─────│──│─────│────┘

┌─────┐──┌─────┐──┌─────┐──┌─────┐──┌─────┐──┌─────┐

│─────│──│─────│──│─────│──│─────│──│─────│──│─────│

│───┌─────┐──┌─────┐──┌─────┐──┌─────┐──┌─────┐────│

└───│─────│──│─────│──│─────│──│─────│──│─────│────┘

┌─────┐──┌─────┐──┌─────┐──┌─────┐──┌─────┐──┌─────┐

│─────│──│─────│──│─────│──│─────│──│─────│──│─────│

Less dense sprites not aligned

In this case we use 12*2*(2*6+4*20/4) = 768 mclk for the ground leaving 256 mclk for the airplanes. This looks much better of what is possible on the Megadrive.

![image alt text](image_11.png)

Afterburner (Megadrive)

This shows how the Megadrive with its fixed pipeline is more efficient but cannot accommodate unplanned setups like Afterburner where tiled layer are useless. On the converse the Panther is less efficient but more flexible.



For a game like Outrun the hardware zoom permits very fluid zoom animation where the Megadrive and SNES have to store all the different sizes in ROM and therefore have noticeable jumps in the zoom animations.

The Panther has another trick to play, RLE encoded sprites in SRAM. RLE sprites are encoded as a sequence of runs, each run contains a color and the number of pixels to draw with that color. For example, a line of 16 pixels of color 3 can be encoded as [(3,16)].

RLE is useful to compress bitmaps with lot of repeated colors. The road in racing game like Outrun is exactly this. So, it is possible to encode each line with about 10 runs. In other words the CPU can very efficiently draw the road.

## Objects List processing

With Objects List processing we mean the update of the Objects List that the CPU does  every frame to animate the graphic.

As already seen, parsing the Objects List can eat a substantial part of the SRAM bandwidth. Is therefore important that the Objects List not only shows the graphic as expected but it also do it in an efficient way. Let’s see why with an example.

We want to simulate a tile-layer using 12 rows of 20 Objects. Each Bitmap Objects is 16x16 pixels large. We create an Object List with 240 Bitmap Objects by linking them from left to right from the top to the bottom.

To parse a Bitmap Object definition requires 4 read accesses and 2 write accesses to SRAM (2 mclk/access) if the Bitmap Object is visible on the current scanline or 1 read access if not visible.

To fetch 4 bit per pixel bitmap data from ROM requires 4 mclk/word, a word is 16 bit and contains 4 pixels. In other words,1 mclk/pixel. This is the same time needed to write on the front line buffer. Since this happens in parallel we can ignore it.



The scanlines composing the first row are completed in 20*(6*2) + 20*(16/4*4) = 240 + 320 = 560 mclk. This sums the mclk needed to parse the definitions and fetch the pixels.

The second row requires 20*(1*2) + 20*(6*2) + 20*(16/4*4) = 40 + 240 + 320 = 600 mclk. This sums the mclk needed to parse the definitions for the first row (nothing visible) and the second row (all 20 visible) then fetch the pixels. So, the formula is

mclk(row) = 560 + 40*(row-1)

At row 12 we need 1000 mclk to draw a scanline. This leaves just 1024-1000=24 mclk for other layers or sprites, practically nothing.

01: O O O O O O O O O O O O O O O O O O O O : 464 mclk

02: O O O O O O O O O O O O O O O O O O O O : 424 mclk

03: O O O O O O O O O O O O O O O O O O O O : 384 mclk

04: O O O O O O O O O O O O O O O O O O O O : 344 mclk

05: O O O O O O O O O O O O O O O O O O O O : 304 mclk

06: O O O O O O O O O O O O O O O O O O O O : 264 mclk

07: O O O O O O O O O O O O O O O O O O O O : 224 mclk

08: O O O O O O O O O O O O O O O O O O O O : 184 mclk

09: O O O O O O O O O O O O O O O O O O O O : 144 mclk

10: O O O O O O O O O O O O O O O O O O O O : 104 mclk

11: O O O O O O O O O O O O O O O O O O O O : 064 mclk

12: O O O O O O O O O O O O O O O O O O O O : 024 mclk

Effect of Object List parsing on free mclk

This example shows that with a naive approach the Panther cannot compete with the Megadrive or the SNES and that the clever design of Object Lists is at the core of Panther’s programming.

Ideally, every scanline should process only the Objects visible on that scanline. To the extreme this would require a specific list for every scanline. Each list is terminated by a Write Object which points the Object List pointer register to the next list. Off course this is not feasible because requires too much SRAM (an average of 10 Bitmap Objects per scanline require 10*16*200 = 32000 bytes, all the SRAM) and CPU processing.

An intermediate approach is needed, like dividing the screen in horizontal buckets and create an Objects List for each. Branch Objects can be useful to skip parts of the list after a given scanline.

As an exercise, we can apply the Branch Objects to the previous example. Every row starts with a Branch Object that normally points to the first tile of its row, but when the row is no more visible branches to the Branch Object of the next row. The last tile of every row points to a list of sprites.

B1 T1,1 ... T1,N \

...    ...     >→ S1 … SS

BM TM,1 ... TM,N /

To fetch a Branch Object requires 1 read access when not branching and 2 read accesses when branching. So, the formula for the tiles is:

mclk(row) = 560 + 2 + 4*(row-1)

01: O O O O O O O O O O O O O O O O O O O O : 462 mclk

02: O O O O O O O O O O O O O O O O O O O O : 458 mclk

03: O O O O O O O O O O O O O O O O O O O O : 454 mclk

04: O O O O O O O O O O O O O O O O O O O O : 450 mclk

05: O O O O O O O O O O O O O O O O O O O O : 446 mclk

06: O O O O O O O O O O O O O O O O O O O O : 442 mclk

07: O O O O O O O O O O O O O O O O O O O O : 438 mclk

08: O O O O O O O O O O O O O O O O O O O O : 434 mclk

09: O O O O O O O O O O O O O O O O O O O O : 430 mclk

10: O O O O O O O O O O O O O O O O O O O O : 426 mclk

11: O O O O O O O O O O O O O O O O O O O O : 424 mclk

12: O O O O O O O O O O O O O O O O O O O O : 420 mclk

Effect of Object List parsing optimization on free mclk

Much better! But we can do even better by keeping the free mclk constant. The idea is to use the Write Object to change the Object List Pointer register at the end of every row.

W1 B21     B11 T1,1 ... T1,N \

...        ...        ... >→ S1 … SS

WM B2M     B1M TM,1 ... TM,N /

The frame starts with the Object List Pointer register pointing to B11.

B1x normally lead to Tx,1 but when the row x is complete branches to Wx which changes the Object List Pointer register so it points to B1x+1. Then B2x always branches back to Tx,1 as a sort of "return" statement. We assumed that the Object List Pointer register is read by the hardware only when the frame starts and has not an immediate effect on the parsing. The formula is:

mclk(row) = 2 + 560 = 562         ; on the first 15 scanlines of a row

mclk(row) = 4 + 4 + 4 + 560 = 572 ; on the 16th scanline of a row

This should give an idea of the flexibility and complexity that an Object List can reach. Other ideas are using the Copy Object or the Write Object to modify the Object List dynamically.

Another matter that complicates Objects List processing is the fact that the Panther Chip rewrites at every scanline the Bitmap Object description of those visible on that scanline (data and Y size fields). Therefore before the begin of a new frame the Bitmap Object descriptions must be restored. This can be done by the CPU or by the Object List using the Write Object or the Copy Object.

If this is not enough, horizontal clipping of Bitmap Object is important to no waste time rendering pixels not visible. The CPU has to clip the Bitmap Objects by correctly configuring their definition.

# Techniques

techniques to better use the hardware

tiles prepared + synthetized in SRAM

Buffer in 32 kb without dual buffering by following the raster a la battle squadron or work on the Vblank

3d games in 32 kb using RLE for fast filling

Dma to rewrite object list parts and implement tile map 16*16 without branch objects

## 3D games

The CPU on the Panther, when it has the bus, can manipulate graphic data directly and conveniently (linear buffer, chunky format (NOTE:  TODO: explain chunky)). This is very important, for example, for 3D rendering.

Off course here we are talking about a primitive form of 3D rendering based on flat filled polygons.

How it was the competition?

The Sega Megadrive has 64KB of SRAM dedicated to the 68000 @ 7,6 Mhz where it can work at full speed with still bandwidth to spare. Anyway when it is time to manipulate graphic data it has to access the VRAM, which is not mapped on its address space, via IO ports with wait states. The graphic data is not organized in a linear fashion so 3D is harder.

The Nintendo SNES has a 65816 CPU @ 3,58 Mhz and around 1.7 MIPS (if efficient as the 6502). Anyway, compared to the 68000, it has less registers and they are 16 bit instead of 32 bit. An additional problem is that it bottlenecked by a 8 bit data bus. Like the Megadrive VRAM must accessed via IO ports and the graphic data layout is even less favorable since it is planar.

Indeed either the SNES and the Megadrive had 3D games only because special game cartridges with RISC CPU and a linear memory mapped frame buffer where created.

On the paper the Panther could have had good 3D games without special cartridges. Except for one detail, it has just 32KB of SRAM.

Can be avoid to wait vblank to switch buffers?

# HW Comparisons

compare with competitors

functioning, components

# Retrospective

It is looks to me that the Flare 2 team was internally pushing for the Jaguar since it was their baby and the Panther was more a work for hire. In some sense the Panther was the son of nobody.

The Panther schematics and programming manuals shows it has many shortcomings probably due to the unfinished state and the pressure the keep the price very low.

Comparing the bill of material is clear that the Atari Panther was cheaper to produce than the Megadrive and SNES. It’s power is near to the Sega Megadrive, sometimes better, sometime worse. For sure it was weaker than the Nintendo SNES.

Porting games from other consoles or personal computers is not simple given the system’s peculiarities. This would had reduced the amount of titles available.

In the end, without further development the Panther was too weak and different to compete with the Sega Megadrive and the Nintendo Super NES even if cheaper and with hardware zooming. Killing it was a good choice.

The Atari Panther looks like as an evolution of the Atari 7800 an 8 bit machine also designed around the concept of Object List.

Unfortunately the Panther has no concept of tiles and makes it less efficient than other consoles. This also makes ports more complex.

In my opinion, the console was not fully developed when the project was cancelled. This is evident from the use of an off the shelves audio chip (the ES5505) and the amount of glue logic needed to interface it and sample the joysticks. It is quite possible that with more development time the glue logic and maybe a custom audio processor would have been integrated in another custom chip greatly reducing the motherboard complexity.

The limited amount of SRAM and the absence of layers and tiles makes difficult to port games from home computer or other consoles. This would had severely impacted the amount of games available on the system.

The Jaguar was released 30 months after killing the Panther. It would have been better to release a slightly better Panther around mid ‘92. This means 1 year to improve the hardware and gain traction among developers. Not all the developers were fine with Nintendo or Sega terms and a third more "indie" console could have had its place if porting games from Amiga or PC was very easy.

The Jaguar release could have been delayed of 1 year and been CD based and with Panther compatibility. None of the 5th generation consoles was back compatible this could have been a big plus. The Panther could also had granted to Atari a better cashflow and a bigger budget for the Jaguar release.

# In a parallel universe

## A better Panther

256px resolution

…

SRAM prices 2002 jameco, ftp://[ftp.jameco.com/Archive/PreviousCatalogs/211catalog.pdf](ftp.jameco.com/Archive/PreviousCatalogs/211catalog.pdf), page 4

In 2002 at small distribution Panther SRAM 8KB 120ns chips cost $2.15 each.

SRAM 32KB 15ns costs $2.69 each.

So, in 2002 equip the Panther with 128KB of 15ns SRAM would had cost 25% more.

6264P-12 	8k x 8 	120ns (64kB) CMOS 		2.15

6264LP-70 	8k x 8 	70ns (64kB) LP CMOS 		2.49

6264LP-10 	8k x 8 	100ns (64kB) LP CMOS 		2.55

62256LP-70 	32k x 8 	70ns low power CMOS 		4.49

62256LP-10 	32k x 8 	100ns low power CMOS 		3.25

62256LP-12 	32k x 8 	120ns low power CMOS 		3.49

ATT7C199P-15 	32k x 8 	15ns high speed CMOS (.3"W) 	2.69

## A different end

I argue that a release in late 91, when the SNES was just released, a bill of materials more in line with the Megadrive and SNES, a hardware more adapt to port games from the still hot Commodore Amiga and PC it could have been a successful product.

Indeed European game developer struggled to adapt to the strict publishing rules enforced by Sega and Nintendo. A third console with more liberal publishing rules and an architecture more adapt to port games from home computers would had some traction.

The Atari Jaguar could have been released in late 94 with a CD, more memory bandwidth and more developers competing with the Saturn and Playstation.

--------------------------------------------------------------------------------------------------------------------------

SRAM costs history jameco

Gates count for snes and Megadrive

Able to speed up drawing if bitmap in SRAM

256x200 gfx mode

New ideas:

* 32Mhz SRAM, interelaved access of 4 sybsystems. ObjList parser, ObjRender, linebuffer&clut read. Inner chip remains 16Mhz. Stencil buffer in PC. No GS. Tristates chip for ROM, 68K decoupling. Audio 8ch 8bit DMA. Option to render line by line on a frame buffer. Multi pass.

* 64KB SRAM for framebuffer and easier ports

* Serial joypads

* 1 cycle ROM access with tristate: instead of give Addr at cycle N and read Data at cycle N+3, give Addr at cycle N and read result of Addr submitted at cycle N-4

* 16 Mhz SRAM with buffers and CLUT and stencil in SRAM. Ability to draw in a frame buffer without waiting new scanline while the scanlines are fetched. E.g. 25Hz refresh but double the sprites.

# Hardware overview

## Competition

### 1988 RAM shortage

[https://tedium.co/2016/11/24/1988-ram-shortage-history/amp/](https://tedium.co/2016/11/24/1988-ram-shortage-history/amp/)

Influenced design, solved in 91

### Megadrive

189 pounds in 1990 [https://segaretro.org/History_of_the_Sega_Mega_Drive#Europe](https://segaretro.org/History_of_the_Sega_Mega_Drive#Europe)

Released in 1988 in Japan.

Released in 1989 in USA. Atari was offered to distribute it. Judged too expensive. [https://segaretro.org/History_of_the_Sega_Mega_Drive#North_America](https://segaretro.org/History_of_the_Sega_Mega_Drive#North_America)

Contains [https://www.console5.com/wiki/Genesis](https://www.console5.com/wiki/Genesis)

* 68000 8 MHz

* Z80 4 MHz

* Ym2612

* SRAM 2*32k*8@100ns 64k

* VRAM 2*64k*4@100ns 64k

* ASRAM 8k*8@150ns 8k

* 4 custom chips

* RGB encoder

* 4 misc chips

### Snes

150 pounds in 1992

[https://en.m.wikipedia.org/wiki/Super_Nintendo_Entertainment_System#Launch](https://en.m.wikipedia.org/wiki/Super_Nintendo_Entertainment_System#Launch)

Released in 1990 in Japan

Contains [https://www.console5.com/wiki/SNES](https://www.console5.com/wiki/SNES)

* Custom 65816 @3.58 MHz

* Custom 128k*8 DRAM

* Custom DSP

* Custom Audio generator

* Dac

* Audio RAM 2*32k*8@120ns 64k

* Video SRAM 2*32k*8@120ns 64k

* 2 custom video chips

* RGB encoder

* 5 misc chips

# System critic

Go deeper in how the system works and what are the pros and cons.

Challenges from the point of view of a developer.

When bitmap is in SRAM it can fetch 32 bit in 2 mclk anyway it cannot take advantage of this because main bus cannot be accessed by the CPU and Line buffer can only accept 1 pixel per mclk.

No need to copy gfx from ROM to VRAM like SNES and Megadrive.

The Panther chip contains a Display List Processor. It is a type of sprite engine. It continues the idea implemented on the Atari 7800. It processes an Object list. The Object list is a linked list of Objects. Each object is composed of few 32 bit words (e.g. 4) and describes a sprite included the pointer to the graphic data and the link to the next Object. The Object list is processed from the start at every scan line. If an Object appears in the current scan line the its graphics data are copied from the ROM in the line buffer otherwise it is skipped. If similar to the Jaguar then Object can have any size and they can be in any number. What limits them is the SRAM size and the number of cycles in a scanline. If similar to the Jaguar there is a "smart" Object that depending on the scanline can change the Object List pointer. Something similar to the Copper on the Amiga.

The sound section contains 8 buffer-type chips, 1 PLA, 1 8x8K SRAM, 1 DAC, 2 OpAmps and the sound chip (Ensoniq ES5505 OTIS). The 74244 on the A bus means that the Sound chip cannot control the A bus and therefore it can only read/write the 8KB of dedicated SRAM. The sound RAM can be accessed by the 68000 and the Panther Chip. Probably the Panther Chip can copy data from ROM to sound SRAM.

The joypad section contains 4 discrete chips (buffers) to sample the joypads and allow the read via the bus.

Limitations:

* Just 32KB of RAM

* There are just 32 entries in the CLUT

* 68000 wastes 94% of the bandwidth when working outside the vertical blank

* The Object List is not on a specific isolated RAM (E.g. like on the Neo Geo). This means that if there are many Objects on screen then lot of bandwidth is wasted just scanning the Display list and very few actually drawing the sprites.

* No tiles layers. This means that also the background has to be implemented with sprites.

* Object list is modified by the Panther chip. The CPU has to restore the height value. This an additional complexity for the 68k.

* Assuming it is similar to Jaguar, then: let’s say that an object is like a Scaled Bitmap Object then 6 accesses are needed to read and 2 to update the object (height and remainder). If done more efficiently I think 4 reads and 1 write are enough.

* Some post in AtariAge say 125ns is the cycle time of the Panther chip. I think the 35ns of the SDK are the correct ones because:

    * 32KB was a very small amount even for 1991 (some 7800 cartridge had this onboard). It make sense to have such small amount if it was very expensive and 35ns SRAM was expensive at the time.

    * Why mount 35ns SRAM on the SDK when 120ns was enough?

    * Why have a 32Mhz quartz (31.5ns cycle) on the system?

* 32 Mhz, 31,25ns pixel clock. 2048 clock per scanline (64us).

* We said every Object requires 8 cycles to be processed from the SRAM.

* Read a word from a 375ns ROM requires 375/31.25=12 cycles.

* Read a 16 pixel 16 color sprite therefore requires 12*4 + 8=56 clock. 2048/56=36.5 sprites per scanline. This means an 1.83x overdraw.

* Unfortunately at every scanline the Object Processor has also to process Objects not visible on that scanline. Consider an implementation where we have 1 background layers composed of 16x16 sprites/tiles. We don’t want them skipped so we put these 20*10=200 sprites on the head of the list. This means that when the raster reaches the last 16 scan lines then the the first 180 sprites of the list are not needed and are skipped. This nevertheless consumes 180*8 clocks, leaving 2048-1440=608 clock free. This amounts to 608/56=10 16 px sprites, this means that, at that scan lines, the Panther chip will not have time to draw the layer. Pretty lame.

* Now redo the same but using Branch Object. BO permits to skip parts of the DL depending on the scanline. We can arrange BG tiles/sprites in strips of 20 after a BO.
On the last 16 scan lines, the Panther will have to skip 9 BO (9 read accesses). Read the last one. Process the 20 objects composing the layer and finally process the "real" sprites.
![image alt text](image_12.jpg)
This means 2048-10=2038. 2038/56=36. This is a 1.81x overdraw. 1 layer plus 16 16px sprites (about 262 px) per scan line. This is better. Indeed will likely have more than 16 sprites on screen so more cycles will be wasted processing objects not visible. One layer and less than 16 sprites per line then. Not very good for 1991.

* This shows that DL organization can have a big impact on the performance. We went from not being able to draw 1 layer to Megadrive level.

* What if SRAM was 125 ns like someone said in AtariAge? Then we have to multiply Object processing cycles by 4.
2048-40=2008. Sprite fetch with 375ns ROM gives 4*12 + 8*4=80. 2008/80=25.1. This is a 1.25x overdraw. The system can barely draw the background layer and 5 sprites (81 px). If we count an overhead for not displayed objects then it cannot draw even the layer.

* What if the ROM is 200ns like the one found on the Neo Geo and Megadrive? 200/31.25=6.4. So a 16px sprite will require 4*7+8=36. 2048/36=56.8. This is 2.84x overdraw. Not too bad.

* Megadrive has 2 layers, max 80 sprite on screen and 20 per scanline. Can we match it? We can afford 2 layers and at the same time is rare that the foreground layer totally covers the background layer. We can say that in average half of the objects/tiles in the foreground layer are not present. This means 30 tiles per line. 2048-10-(30*36)=958 free clocks. Process 60 not visible objects leaves 958-(60*8)=478 free clocks. 478/36=13.2 sprites per scan line. Somehow in pair and a bit weaker than the Megadrive. With 5 accesses per object we arrive to 22 sprites per scanline. In pair with Megadrive.

* Nec PC engine used 140ns ROM. In this case: 140/31.25=3.4. We’ll use 4. We have 35 sprites per scanline and 52 sprites with 5 accesses per object.

* I can speculate that the SDK contains 100ns fake ROM because permitted to emulate all the possible ROM speedgrade.

* What if the ROM is 100ns like in the DevKit? 100/31.25=3.2. If we take the safe route we’ll use 4 like the PC Engine. If we use it with 3 cycles (93ns) then we can have 47 sprites per scanline or 72 if we use 5 accesses per object.

* With average ROM speed (200ns) and clever use of DL it was in pair with the Megadrive but with zoom and direct access to ROM gfx.

* With fast ROM speed (140ns, like PC engine) and compact object representation, it was able of dual layer, 60 sprites on screen and 52 sprites (840 px) per scanline. This starts to be competitive with Neo Geo and makes game like Afterburner or OutRun extremely doable.

* With 100ns ROM then dual layer and 70 sprites on screen even on all on the same scanline.

* The Panther was able to read sprites from ROM the programming it was significantly easier than on MegaDrive where you have to continuously DMA data from ROM to VRAM. This could compensate the complexity of creating efficient display lists.

* There was not latch on the cartridge, bus was locked by the ROM even if 12 cycles were needed. A latch could have free the bus while the ROM was retrieving the data.

* It looks like the joypad and sound section contain a lot of discrete components. With something like 4 buffer-type chips it would have been possible to implement the Amiga style interleaved RAM access. Wasted bandwidth would have been 2+8/8+8=10/16=63% . With some more buffer chips it would have been possible to cut of the slow ROM from the bus when waiting for data. Less powerful chips like the AY-3-8910 [https://en.wikipedia.org/wiki/General_Instrument_AY-3-8910](https://en.wikipedia.org/wiki/General_Instrument_AY-3-8910) could have been used to provide FM audio (no PCM) plus joypad reads without all the glue logic. The 8KB sound RAM chip could have been used for the 68000. PCM could have been easily implement in the game shifter Chip.

* The shared bus means the SRAM is not used most of the time because a ROM access is in progress. SSSSS___r___r__..._rS.

# Performance considerations

Based on ROM wait states, define expected performance. Compare with consoles, home computers of the era and coin-ops.

# A better Panther

It was rushed.

Define how it could have been improved.

Version 1: same cost

Version 2: more cost

## Stencil buffer

* The stencil buffer is a feature of 3D GPUs. In some implementations it is a 1 bit buffer that blocks the write of the corresponding pixel when set to 1. It is can be set to 0 at the beginning of the drawing and to 1 when the pixel is written for the first time.

* If the scene is drawn from the front to the back then pixel written multiple times will result in a read of the source buffer only the first time. This means that even if the scene has a overdraw of Nx the source buffer will be accessed just and always 1x.

* This is particularly powerful when the source buffer is very slow compared to the destination and stencil buffer.

* When objects are drawn front to back it could happen that in a very complex drawing there is no more time to draw the last objects. This can leave, for example, an empty background and this can be aesthetically not acceptable. In this case is useful to disable writes on the stencil buffer, write the background, re-enable writes on the stencil and start draw from front to back.

* This also explain why the stencil buffer is often implemented with dedicated bits and not with comparators that test if a pixel has a value different from 0.

## Stencil buffer on the Panther

* This technique can be used on the Panther by adding 1 bit to the line buffer.

* This is really suitable for the Panther architecture since it has very fast line buffers but very slow ROM.

* **Hypothesis 1**: there is space on the GS to accommodate the stencil buffer and its logic.

* **Hypothesis 2**: we can enlarge the pixel bus (PB) between the PC and the GS from 5 to 16 lines.

* The PC can read from the GS 16 bit of the stencil buffer starting from position x in a single cycle.

### Does the Panther originally included a stencil buffer of sort?

* No, the GameShifter schematics show it doesn't. Already from the first sheets is clear this is not possible since the pixel bus pads are input only. The PC can only write pixels into the GS, it  cannot read. Therefore it cannot contain a stencil buffer as the one described above.

### Is the stencil buffer something unknown when the Panther was designed?

The Panther was designed between 1988 and 1991 in parallel with the Jaguar. The z buffer, a more powerful concept than stencil buffer, was known since 1974. Matrox produced a graphic card with z buffer, the SM-640, in 1987. The Jaguar has a z buffer. So the z and stencil buffer was a concept well known by who designed both the Panther and the Jaguar.

### How it works?

* The Panther Chip needs to query the stencil before fetching data from ROM.

    * First row is the main bus

    * Second row is the GS bus

    * S is SRAM access to read and update the object

    * R is the ROM access with wait states

    * t is the stencil test. Note that the main bus is free during this slot

    * p is a pixel written in the LB

* The following access pattern shows a object processed where quads 1,2,3,6 require ROM access.  Quads 4,5,6,7,8 not. Then a new object is processed. First t is for quads 1,2,3,4. Second is for 5,6,7,8.

* Main: SSSS-RRRR----RRRR----RRRR-----RRRR----SSSSS-

* GS  : ----t----pppp----pppp----ppppt----pppp-----t

* Quad: |____1_______2_______3________6________|____

* Note how writing pixels to LB stalls the stencil test postponing the ROM access. This is leaves the already slow ROM unused half the time.

* Can it be improved?

### Pipelining

* **Improvement**: ROM access pipelined with LB writes

* Main: SSSS-RRRRRRRRRRRR-----RRRRSSSSS-

* GS  : ----t----ppppppppppppt----pppp-t

* Quad: |____1___2___3________6____|____

* Note how the ROM is fully used except after a skipped access but that would have been a useless access, so it is OK. Plus in this case the main bus is free.

* Note how pipelining allows to start processing new object while last quad is still in writing. So no penalty when starting a new object.

* Note how after a rejected ROM access the bus remain unused.

* Can it be improved?

### Stencil test precedence

* **Improvement**: Writes on the LB can be postponed after the stencil test

* Main: SSSS-RRRRRRRRRRRR-RRRRSSSSS-

* GS  : ----t----pppppppptpppppppp-t

* Quad: |____1___2___3____6____|____

* Note how doing the second test as soon as possible allows to start another ROM access without waiting the end of the writing of the quad 3 in the LB. In this way the busses are fully utilized except at the beginning of the object processing.

### More access patterns

* The following access pattern shows a object processed, shrinked at ¼ where pixels 1,2,3,17 require ROM access other pixels not. Then a new object is processed. First t is for pixels 1 to 16.

* Main: SSSS-RRRRRRRRRRRR-RRRRSSSSS-

* GS  : ----t----p---p---tp---p----t

* Quad: |____0___0___0____1____|____

* Quad: |____1___2___3____6____|____

* The following access pattern shows a object processed where all the 320 pixels doesn’t require ROM access. Then a new object is processed.

* Main: SSSS--------------------SSSSS-

* GS  : ----tttttttttttttttttttt-----t

* Quad: |________________________|____

* Note: how shrinking or zooming doesn’t impact the number of cycles.

* Note: how just 320/16=20 tests are needed to cover all the LB

### What is the expected performance?

* Any object requires header processing (5 cycles) and a test every 16 pixels drawn.

* If any pixel is actually drawn it requires a additional 1 (no shrink) to 4 (¼ shrink) cycles even if it is transparent.

* There are 320 drawable pixels

* Transparent pixel will cause another write

* Zoomed objects save ROM accesses since less data is needed to fill the line buffer.

* Shrinked objects use more ROM accesses, worst case is ¼, here every ROM access wastes 3 pixels.

* Simulation 1

    * Assume: 100 objects, 128px wide, shrinked ¼, 20% transparent source pixels

    * 100*5=500 - object header processing

    * 128/16*100=800 - stencil tests

    * 320*4=1280 - ROM access cycles for opaque pixels

    * 128*100*0.2*4=10240 - ROM access cycles for transparent pixels. worst case, tr. pixels never skipped thanks to stencil buffer.

    * Total: 12820 over the 2048 available

* Simulation 2

    * Assume: 40 objects, 20 on the scanline, 128px wide, shrinked ¼, 20% transparent source pixels, stencil buffer decimate 80% of transparent pixels.

    * 40*5=200 - object header processing

    * 128/16*20=160 - stencil tests

    * 320*4=1280 - ROM access cycles for opaque pixels

    * 128*20*0.2*0.2*4=410  - ROM access cycles for transparent pixels

    * Total: 2050 over the 2048 available

* Simulation 3

    * Assume: 100 objects, 48 on the scanline, 128px wide, some shrinked ¼, some normal, some zoomed 2x, 20% transparent pixels, stencil buffer decimate 80% of transparent pixels, 50% pixels from normal, 25% from shrinked, 25% from zoom

    * 100*5=500 - object header processing

    * 128/16*48=384 - stencil tests

    * 80*4=320 - ROM access cycles for opaque pixels

    * 160/4*4=160 - ROM access cycles for opaque pixels

    * 80/8*4=40 - ROM access cycles for opaque pixels

    * 128*48*0.2*0.2*4=984 - ROM access cycles for transparent pixels

    * Total: 2387 over the 2048 available

* Simulation 4

    * Assume: 100 objects, 48 on the scanline, 128px wide, no zoom or shrink, 20% transparent pixels, stencil buffer decimate 80% of transparent pixels

    * 100*5=500 - object header processing

    * 128/16*48=384 - stencil tests

    * 320/4*4=320 - ROM access cycles for opaque pixels

    * 128*48*0.2*0.2*4=984 - ROM access cycles for transparent pixels

    * Total: 2188 over the 2048 available

* Simulation 5

    * Assume: 128 objects, 96 on the scanline, 16px wide, no zoom or shrink, 20% transparent pixels, stencil buffer decimate 80% of transparent pixels

    * 128*5=640 - object header processing

    * 16/16*96=96 - stencil tests

    * 320/4*4=320 - ROM access cycles for opaque pixels

    * 16*96*0.2*0.2*4=246 - ROM access cycles for transparent pixels

    * Total: 1302 over the 2048 available

* Simulation 3 shows it can run games like Galaxy Force 2.

* Simulation 4 shows it can do the same when no zoom/shrink is used.

* Simulation 5 shows it can match Neo Geo specs without zoom leaving 37% cycles free for shrinking.

* Note: cycles used for stencil tests leave the SRAM free

* Note: ROM wait states use a considerable part of the cycles.

## Detachable ROM

* If the ROM is detached from the main bus then it can be used to access SRAM

* If the ROM requires N cycles then only the first and last are required to use the main bus.

* The first to sample the address lines

* The last to sample the data lines

* It goes from

* Main: SSSS-RRRRRRRRRRRR-RRRRSSSSS-

* GS  : ----t----p---p---tp---p----t

* Quad: |____0___0___0____1____|____

* Quad: |____1___2___3____6____|____

* to

* Main: SSSS-R--RR--RR--R-R--RSSSSS-

* GS  : ----t----p---p---tp---p----t

* Quad: |____0___0___0____1____|____

* Quad: |____1___2___3____6____|____

* 5 or 6 74xxx chips are required

## More colors

* GameShifter Signals

    * I - Line buffer B address - 9

    * I - Line buffer E address - 9

    * I - Pixel data - 5

    * O - RGB analog - 3

    * O - RGB digital - 18

    * O - BLANK_ - 1

    * IO - Main bus Data - 16

    * I - Main bus address - 6

    * I - UDS_ (CPU upper byte) - 1

    * I - LDS_ (CPU lower byte) - 1

    * I - RW (CPU read/write) - 1

    * I - RAM_ (CPU enable CLUT, clock) - 1

    * ? - DEN_ - 1

    * I - LD_ - 1

    * I - EI_ - 1

    * I - WP - 1

    * I - SC (system clock) - 1

    * I - VSS (power) - 3

    * I - VDD (ground) - 3

### The problem

* With 5 bits per pixel (32 colors CLUT) the Panther graphics results less colorful than the Megadrive. Not even comparable to Neo Geo or arcades. More like a very fast zooming Amiga.

* If there are just 5 bits per pixel, we can guess, it is because the gates in the GS have been all used.

* More bits require more gates and in turn a bigger and costly chip.

* Can we have a bigger CLUT without increasing the cost of the console?

### The idea

* Since there are lot of free SRAM cycles, thanks to the stencil test, we can use them for moving the back line buffer and the CLUT on the SRAM.

* Having enough free cycles is not enough. Producing the video signal requires extreme precision. It must be possible to access the main bus without wait states. In other terms, read the line buffer and the CLUT has the maximum priority over all the other uses.

* Detaching the ROM gives us also this aspect. It is now possible to delay the first or the fourth ROM cycle if the SRAM is needed for the video signal generation. Without a detachable ROM the bus remains locked for 4 cycles and blocks the video signal generation. Therefore detachable ROM is a prerequisite to move the CLUT and back line buffer in SRAM. Off course, it also free more bus cycles.

### A bigger front line buffer

* CLUT and the back line buffer are removed from the GS

* This frees 5*320 + 18*32 bit. Since also the address decode logic is  removed the  saving in term of gates is even bigger.

* It is very likely that a single 10 bit front line buffer with 1 bit stencil can be accommodated

* This clears the hypothesis 1 introduced with the stencil buffer.

### A bigger CLUT

* The CLUT in SRAM is no more constrained by the GS gates count. With 10 bit per pixel in the line buffer a CLUT with 2^10=1024 entries is needed.

* 16 bit per entry means 2KB of SRAM. Note that not all the entries have to be used.

* In simpler words with this modification we go from 32 colors among 256K shades to 1024 color chosen among 65K shades.

* This specs are Neo Geo or Arcade class.

### Data paths

* How the front line buffer within the GS is copied on the back line buffer in SRAM?

* The front line buffer is copied to SRAM during the hblank using a 32 bit datapath between P and GS chips.

* This 32 bit data path is obtained by extending the pixel bus from 5 to 16 lines.

* The 16 bit pixel bus also allows the 16 bit stencil reads described before.

* The GS already has access to the lower 16 bit of the main bus. This was used by the CPU to read and write the CLUT.

* During the copy, the PC connects the pixel bus to the high 16 bit of the main bus (remember this is already implement to allow the CPU to access the upper SRAM bank) and the low 16 bits are directly provided by the GS. This is the 32 bit data path.

* Drawing ...

* How obtain this additional 11 lines without using a bigger package for the GS chip?

* Without CLUT its address lines and control lines aren't needed. This frees 9 pins.

* Without back line buffer its address lines are not needed. This frees at least 9 more pins.

* So with 18+ pins free we can allocate 11 to the wider pixel buffer and spare 7+

* This clears hypothesis 2 introduced with the stencil buffer.

### Timings

* The back line buffer in SRAM packs 3 pixels in 32 bits (3*10=30<32). This means 428 bytes on SRAM.

* With this packing the copy requires 320/3=110 cycles across a 32 bit data path.

* A scanline is 64 ms and the active part (where the 320 pixels are displayed) is 52 ms. Therefore, the hblank (borders plus beam fly back) lasts 64ms - 52ms = 12 ms.

* This means that 12000/31,25=384 slots are available during hblank.

* This is enough to copy the front line buffer in SRAM during the hblank.

* During the active line the PC reads the back line buffer and CLUT from SRAM. The back line buffer is read with 32 bit accesses. The CLUT is read with 16 bit accesses, one per pixel.

    * 1 access every 13 or 14 cycles for line buffer read. 140*3/31.25=13.44

    * 1 access every 4 or 5 cycles for CLUT read. 140/31.25=4.48

### The video DAC

* If we removed the CLUT from the GS, where the 16 bit CLUT value is converted in a RGB analog signal?

* This still happens on the GS.

* Since the GS is connected to the lower 16 but data bus it can observe the CLUT values and convert them in analog RGB.

* The PC generates the CLUT address on the main bus, the GS reads the value returned by the SRAM.

### A smaller GS

* As noted before by moving the back line buffer outside the GS we have 7+ free unused pins.

* The GS has 18 pins dedicated to the digital RGB output

* These are not used in the Panther console.

* If drop this feature we have 25+ unused pins.

* It is possible to go from a 84 pins package to a 64 pins package

* This can reduce the cost of the GS. The savings can be spent on other aspects of the console.

### The new performance

* The additional accesses used by back line buffer and CLUT just shift the the other lower priority accesses.

* So we can just subtract them to the 2048 cycles on a scanline.

* 2048-110-110-320=1508

* This means the 73% of the performance described before.

* This is still enough for the scenario 5 (1300 cycles per scanline) which was Neo Geo class without shrinking.

* Simulation 6

    * Assume: 64 objects, 48 on the scanline, 64px wide, some shrinked ¼, some normal, some zoomed 2x, 20% transparent pixels, stencil buffer decimate 80% of transparent pixels, 50% pixels from normal, 25% from shrinked, 25% from zoom

    * 64*5=320 - object header processing

    * 64/16*48=192 - stencil tests

    * 80*4=320 - ROM access cycles for opaque pixels

    * 160/4*4=160 - ROM access cycles for opaque pixels

    * 80/8*4=40 - ROM access cycles for opaque pixels

    * 64*48*0.2*0.2*4=492 - ROM access cycles for transparent pixels

    * Total: 1524 over the 1508 available

* Scenario 6 is still a overdraw of 64*48/320=9.6 which is very good

* Effect of slower ROM …

* Insulate 68000 and ROM from master bus. This also permits to the 68000 to use all the empty access slots leaved by the ROM wait states.

* Remove the OTIS and implement a simple PCM based audio subsystem (a la Paula) in the Panther chip. A new Object type can be introduced to program the DMA for these PCM channels. Two adders plus shifter can be used to average the samples and then emit serially reusing the serial DAC used by OTIS. This saves a SRAM chip and lot of glue logic.

* A chips like the AY-3-8910 [https://en.wikipedia.org/wiki/General_Instrument_AY-3-8910](https://en.wikipedia.org/wiki/General_Instrument_AY-3-8910) (maybe stereo) could have been used to provide FM audio plus joypad reads. This would have saved even more glue logic and reduced the pressure on the audio PCM engine.

* External SRAM for the Palette used also by the 68000. Gates saved on GS used for more bits. 32*18/320=1,8 → 7bits per pixel. This gives more RAM and allow the 68000 to run always.

* If the GS was gates constrained then a new one could have been produced with a single line buffer inside. Two of the GS chips would have been used in parallel. This would have allowed up to 14bits per pixel. Or maybe 10 and leaved a lot of gates to implement the R buffer.

* Sampling joysticks with a single buffer powering one port at time. Less lines and chips

16 MHz, 2+ CLK x pixel, 60ns
BxRCFxxC  BxOCOxOC  xxxCxxxC
BPxPFPxP  BPOPOPOP  xxxcxxxc

Lot of tech details here:

[http://www.sega-16.com/forum/showthread.php?18434-Comparison-of-4th-generation-(-quot-8-16-bit-quot-)-system-hardware/page76](http://www.sega-16.com/forum/showthread.php?18434-Comparison-of-4th-generation-(-quot-8-16-bit-quot-)-system-hardware/page76)

[http://atariage.com/forums/topic/119048-its-1993-youre-in-charge-of-the-jag-what-do-you-do/page-40?&&p=2013542#entry2013542](http://atariage.com/forums/topic/119048-its-1993-youre-in-charge-of-the-jag-what-do-you-do/page-40?&&p=2013542#entry2013542)

[http://www.homecomputer.de/pages/panther.html](http://www.homecomputer.de/pages/panther.html)

[https://www.chzsoft.de/asic-web](https://www.chzsoft.de/asic-web)

[https://www.hillsoftware.com/files/atari/jaguar/jag_v8.pdf](https://www.hillsoftware.com/files/atari/jaguar/jag_v8.pdf)

Fastest ROM speed supported is 5 clock → 188ns

6 → 222ns

8 → 296ns

Slowest 10 clock → 376ns

[http://](http://svn.fpgaarcade.com/svn/ReplayPub/docs/custom/atari/Atari_ASICs/ASIC/4118_GAME_SHIFTER/DWG/pdf/ST4118.pdf)SVNguest:s1fUXdnc3a@[svn.fpgaarcade.com/svn/ReplayPub/docs/custom/atari/Atari_ASICs/ASIC/4118_GAME_SHIFTER/DWG/pdf/ST4118.pdf](http://svn.fpgaarcade.com/svn/ReplayPub/docs/custom/atari/Atari_ASICs/ASIC/4118_GAME_SHIFTER/DWG/pdf/ST4118.pdf) SVNguest s1fUXdnc3a

[http://www.konixmultisystem.co.uk/index.php?id=interviews&content=martin](http://www.konixmultisystem.co.uk/index.php?id=interviews&content=martin)

<<He became a director of Atari in Sunnyvale and he had a project called Panther - It wasn't called Panther when I joined. Panther was the name of the car my wife had just bought, a Panther Kallista and the chip had no name and I wanted to give it a handle - so it was called Panther.

The design and specification had already been started, and they said "somebody's left - here's the concept" and it was only the video part of the chip - there was no sound.

It was a novel video architecture that allowed you to create windows of different sizes and different bit depths. Essentially you didn't have a frame store - it was a composite of frame stores - a kind of smart video frame store. It would have allowed a great deal of sprite style animation. Sprites in general in those days would have been of a fixed size e.g. 16x16. The games looked 'spritey' because of that, this would have been quite an interesting departure. I wasn't keen on it, but I designed it and the chip was built.>>

[http://atariage.com/forums/topic/143757-what-games-on-other-systems-show-the-jaguars-power/page-2?&&p=1751088#entry1751088](http://atariage.com/forums/topic/143757-what-games-on-other-systems-show-the-jaguars-power/page-2?&&p=1751088#entry1751088)

<<The Panther's graphics chip was the predecessor of the Jaguar's Object Processor. The two are very similar in how they operate, although the Panther had a couple of interesting features not in the Jaguar and of course the Jaguar's OP was far more flexible and advanced, and capable of producing many more colors.

Probably the best way to think of the Panther is a slightly weaker Sega Genesis that can display larger, zoomable, sprites. The number of on screen colors is limited like the Genesis. Unlike the SNES, there is no rotation (or "Mode 7"), no 256 color palettes, no 15-bit RGB, and no color blending/semi-transparency effects.

The graphical tour de force for the Panther would be a game like Afterburner.

In terms of CPU power, it should have been somewhat weaker than a Genesis since the CPU could only run while the (bus-hungry) graphics chip was idle. Additionally, the Genesis had 128KB of RAM split 50/50 between the CPU and graphics, while the Panther had only 32KB and was shared between graphics and CPU.>>

[http://www.gamingmagz.com/from-the-past/secret-specs-of-ataris-16-bit-panther.html#lightbox/0/](http://www.gamingmagz.com/from-the-past/secret-specs-of-ataris-16-bit-panther.html#lightbox/0/)

<< These chips include a feature known as hardware scaling, which means that game designers can make objects appear to zoom larger or smaller on the screen. Although scaling can be accomplished in other ways without this feature, it’s generally faster and smoother when included as a built-in function. Atari’s hand-held Lynx has hardware scaling, as does Nintendo’s Super NES. However, sources say the Panther’s graphics chips do not have a similar function known as hardware rotation, which would allow screen objects to be easily rotated. So far, the Super NES is the only home game system with built-in rotation.>>

[http://imgur.com/a/lbdFc](http://imgur.com/a/lbdFc)

[https://segaretro.org/Sega_Mega_Drive/Technical_specifications#cite_note-:File:MB834200A_datasheet.pdf_p.7B.7B.7Bpage.7D.7D.7D-73](https://segaretro.org/Sega_Mega_Drive/Technical_specifications#cite_note-:File:MB834200A_datasheet.pdf_p.7B.7B.7Bpage.7D.7D.7D-73)

200ns ROM on Megadrive

[https://segaretro.org/Sega_Mega_Drive/Technical_specifications#cite_note-:File:MB838200B_datasheet.pdf_p.7B.7B.7Bpage.7D.7D.7D-74](https://segaretro.org/Sega_Mega_Drive/Technical_specifications#cite_note-:File:MB838200B_datasheet.pdf_p.7B.7B.7Bpage.7D.7D.7D-74)

130ns ROM on Megadrive

From [The One issue 35, page 42](https://archive.org/stream/theone-magazine-35/TheOne_35_Aug_1991#page/n41/mode/2up)

8384 colors on screen from 256K. → it can update it’s 32 entries palette at every of the 262 lines (so this is PAL).

This is possible since the palette can be modified from the main bus and the Panther Chip can drive the bus.

83840 sprites on screen. This is 320 sprites per scanline. At 32Mhz there are 2048 cycles in a scanline (64ms). 2048/320=6.4. This says that the Panther Chip can process an Object in the Object list in 6 cycles. Off course this doesn’t count the time needed to render it. So it can rendere 83840 off screen sprites per frame :).

Max Cartridge size is 16MB. This is compatible with the 23 address lines (8M) and 16 bit data bus on the cartridge slot.

Use [http://wavedrom.com/tutorial.html](http://wavedrom.com/tutorial.html) for the diagrams.

[https://archive.org/stream/radio_electronics_1991-01/Radio_Electronics_January_1991#page/n67/mode/2up](https://archive.org/stream/radio_electronics_1991-01/Radio_Electronics_January_1991#page/n67/mode/2up)

6264-150 a 1.40$ come 4164-120

[https://archive.org/stream/radio_electronics_1989-10/Radio_Electronics_October_1989#page/n95/mode/2up/search/static](https://archive.org/stream/radio_electronics_1989-10/Radio_Electronics_October_1989#page/n95/mode/2up/search/static)

much more costly, up tp 10$
