---
layout: post
title: "Thin Outlines From the G-Buffer Under TAA: Tile Classification"
date: 2026-10-11 15:30:00 +0900
categories: [rendering, npr]
hidden: true
---

Outlines cover a small part of the screen, yet every pass of the algorithm
used to run on every pixel, and every pixel paid for the loads that told it
there was nothing to draw. This post covers the tile classification that
makes the drawing passes run only where the records are: a mask of where
the records lie, a list of the tiles within reach of one, and passes
started from that list without the CPU ever learning its length. It also
covers what the work showed about the passes that remained, which ended up
waiting on their own loads rather than doing too much of them. It is part of
a [series]({% post_url 2026-10-11-thin-outline-plugin %}); the passes it
schedules are described in the
[drawing and performance]({% post_url 2026-10-11-thin-outline-drawing-and-performance %})
post.

## Most of the Screen Has No Record

In the static view I measured the flicker in (1280×720 display, 73% screen
percentage, so 935×526 rendering pixels), 3.1% of the rendering pixels hold
a record. A pixel draws from its own records and those up to two pixels
away, so a block of pixels has something to draw only when a record lies
inside it or within two pixels of it. Counted on that frame:

| Block size, with a 2-pixel margin | Empty | Crease records only | Silhouette records only | Both |
|---|---:|---:|---:|---:|
| 8×8 | 86% | 4% | 6% | 3% |
| 16×16 | 80% | 6% | 8% | 6% |

Before the tiles, the passes did not know this. The record pass has to run
everywhere, since a new edge can appear anywhere, but the passes that draw
the outline ran on every pixel too, and each pixel loaded its records and
its neighbours' before it found they were empty. The worst of them was the
drawing after the upscaler, the default. It runs once per *display* pixel,
and at 73% a display pixel overlaps about 1.73 rendering pixels per axis,
about three in all, so it repeated every rendering pixel's loads and fits
three times. Per pass, at base clocks (the GPU locked to its base clock so
that the times repeat) on an RTX 5080, static camera:

| Pass, base clocks | 100%, drawn after the upscaler | 73%, drawn after the upscaler |
|---|---:|---:|
| Record pass | 0.110 ms | 0.062 ms |
| Drawing after the upscaler | 0.080 ms | **0.194 ms** |
| All passes | 0.190 ms | 0.256 ms |
| **In-game** (normal clocks, the whole plugin) | **0.152 ms** | **0.205 ms** |

At 73% the drawing pass was three quarters of the plugin's time, three times
the record pass, and the default mode cost twice what drawing before the
upscaler cost at the same resolution (0.098 ms in-game). These numbers, and
every number in this post until the last section, were measured before the
edge pass existed: the drawing passes then still selected and filtered the
edge themselves, from the records.

## How the Engine Does It

Unreal already does this in several places. Its motion blur classifies tiles
before filtering, and Substrate's lighting and MegaLights do the same with
more classes. The pattern is three small passes before the real work:

1. **Classify.** One thread per tile decides whether the tile needs work.
   The threads of a thread group count their listed tiles in group shared
   memory, one thread reserves that many slots in a global **tile list**
   with a single atomic add, and each thread writes its tile's coordinate
   into its slot.
2. **Setup.** One thread copies the list's length into **indirect
   arguments**: the group counts a dispatch takes.
3. **The work, dispatched indirectly.** The GPU reads the group count from
   the buffer the setup wrote. Each thread group takes the tile at its own
   index in the list and works on that tile's pixels.

The CPU never learns how many tiles there are, so nothing is read back and
nothing waits. Since a thread group works on one tile, all its threads run
the same code on neighbouring pixels.

## The Record Mask

The classify pass needs to know where the records are, and reading them
directly is wasteful: with the two-pixel margin, a tile's reach overlaps its
neighbours', and every rendering pixel would be read by two to four tiles.
So a small pass right after the record pass condenses the records into a
**record mask** first, and every record is read once.

The mask cuts the view into **squares** of 8×8 rendering pixels (117×66
squares at 73%) and stores one 32-bit integer per square. Inside a square
the pixels are paired into 2×2 **blocks**, 4×4 = 16 of them; bit $i$ of the
integer says that block $i$ holds a crease record, bit $16 + i$ that it
holds a silhouette record. The pass runs one thread group of 8×8 threads per
square, each thread reads its pixel's two record sample counts (the sign
tells the type), sets its block's bits, and the group ORs them together:

![An 8 by 8 grid of rendering pixels with thick lines every two pixels, dividing it into 4 by 4 blocks labelled b0 to b15 in reading order. A slanted crease runs down the square: the pixels holding crease records are green, one per row, at columns 6, 6, 5, 5, 4, 4, 3 and 3 for rows 0 to 7, so they fall in blocks b3, b6, b10 and b13. Three pixels in row 6, columns 0 to 2, are red, holding silhouette records, in blocks b12 and b13. To the right, a legend gives the block index as row divided by 2 times 4 plus column divided by 2, says that bit i means block i holds a crease record and bit 16 plus i that it holds a silhouette record, and draws the integer as two rows of 16 cells: bits 0 to 15 with ones at bits 3, 6, 10 and 13 in green, bits 16 to 31 with ones at bits 28 and 29 in red. The integer is 0x30002448](/assets/images/thin-outline/record-mask.svg)

Why sixteen blocks per square and not one bit per square and type: the
margin would then be counted in whole squares, up to eight pixels instead of
two, and more tiles would be drawn for nothing. On the 73% frame, testing
every record pixel exactly lists 2,007 display tiles, the 2×2 blocks list
2,087, and one bit per square would list 2,567, 23% more, which the drawing
pass would pay for. The texture holds one integer per square either way, so
the extra bits cost nothing. The mask pass itself costs 0.012 ms at 100%
and 0.009 ms at 73% (base clocks).

## Squares Are Not Tiles

The mask's squares are in rendering pixels; the **tiles** are 8×8 pixels of
the pass being drawn. Before the upscaler the two grids are the same. After
it they are not: at 73% a display pixel is 0.73 rendering pixels wide, so a
display tile covers about 5.8×5.8 rendering pixels and lies across the
squares' boundaries wherever it falls.

The classify pass, one thread per tile, works out the rectangle of
rendering pixels the tile's display pixels overlap, grows it by the
two-pixel margin, rounds it out to whole 2×2 blocks, and tests the bits of
those blocks in the squares the rectangle touches:

{% raw %}
```hlsl
// The rendering pixels the tile's target pixels overlap, grown by the margin and clamped to the view.
const float2 TileMin = float2(Tile * THIN_OUTLINE_TILE_SIZE) * RenderPixelsPerDisplayPixel;
const float2 TileMax = float2(Tile * THIN_OUTLINE_TILE_SIZE + THIN_OUTLINE_TILE_SIZE) * RenderPixelsPerDisplayPixel;
const int2 FirstPixel = max(int2(floor(TileMin + THIN_OUTLINE_MIN_OVERLAP)) - THIN_OUTLINE_TILE_MARGIN, 0);
const int2 LastPixel = min(int2(ceil(TileMax - THIN_OUTLINE_MIN_OVERLAP)) - 1 + THIN_OUTLINE_TILE_MARGIN, ViewSize - 1);

// The 2x2 blocks covering those pixels (the reach grows by up to a pixel), and the 8x8 squares covering the blocks.
const int2 FirstBlock = FirstPixel / THIN_OUTLINE_MASK_BLOCK_SIZE;
const int2 LastBlock = LastPixel / THIN_OUTLINE_MASK_BLOCK_SIZE;
const int2 FirstGroup = FirstBlock / THIN_OUTLINE_MASK_BLOCKS_PER_GROUP;
const int2 LastGroup = LastBlock / THIN_OUTLINE_MASK_BLOCKS_PER_GROUP;
```
{% endraw %}

(`THIN_OUTLINE_MIN_OVERLAP` is 0.001 rendering pixels: an overlap thinner
than that is rounding of the pixel ratio, which the drawing pass skips
too.) The rounding to blocks can add up to a pixel on each side, so the
reach is two to three pixels, never less than two. For one tile at 73%:

![A grid of rendering pixels from x equals 28 to 58 and y equals 22 to 42, with thick lines every 8 pixels marking the squares of the record mask at x equals 32, 40, 48 and 56 and y equals 24, 32 and 40. A purple square, display tile (7, 5), lies across the boundaries from x equals 40.9 to 46.75 and y equals 29.2 to 35.1, divided into 8 by 8 thin purple display pixels each 0.73 rendering pixels wide. A blue rectangle marks the rendering pixels the tile overlaps, x 40 to 46 and y 29 to 35; a dashed blue rectangle grows it by 2 pixels to x 38 to 48 and y 27 to 37; a lightly shaded dotted rectangle rounds that out to whole 2 by 2 blocks, x 38 to 49 and y 26 to 37. The six squares this touches, three across and two down, are shaded yellow: their integers are the ones read. A green pixel at x 49, y 31 holds a crease record three pixels from the overlapped pixels, outside the 2-pixel margin but inside the rounded blocks, so the tile is listed. A note gives the conditions: 73%, 1280×720 display, 935×526 rendering](/assets/images/thin-outline/tile-reach.svg)

How many squares a tile reads depends on the screen percentage: at 50% a
display tile is 4×4 rendering pixels, and with the margin it touches one or
two squares per axis; at 73% two or three; at 100%, where the grids line up,
the 12-pixel rectangle always touches three. Nothing but integers is read.
If any tested bit is set, the tile goes into the list, its entry packing the
tile's x and y in 14 bits each and the record types found in the top bits.
At 73% the list held 2,087 of the 14,400 display tiles (14.5%).

Since the edge pass was added (the
[drawing and performance]({% post_url 2026-10-11-thin-outline-drawing-and-performance %})
post), the drawing after the upscaler no longer reads records but one line
per rendering pixel, which the edge pass reconstructs from the records. The
margin is still the right one: a pixel's line comes from records at most
two pixels away, so a display tile with no record within reach has no line
within reach either. The edge pass itself runs from a list of its own,
built the same way over rendering tiles. Drawing before the upscaler needs only that
one list, for every pass.

## One List, One Dispatch

My first version kept three lists, one per class (crease records only,
silhouette records only, both), with a dispatch each, so that each class
could compile without the other type's code. It was correct, and at 73% it
cut the drawing pass from 0.191 to 0.096 ms at base clocks, less than half
what the tile count promised. At 100% the three dispatches took 0.0117,
0.0118 and 0.0119 ms, whatever their tile counts. Why they all took the same
time is the reason the plugin now has one list.

A GPU runs threads in **warps**, 32 at a time in lockstep; a thread group of
8×8 threads is two warps. The RTX 5080 has 84 multiprocessors, each holding
up to 48 resident warps, 4,032 in all. A texture load takes hundreds of
cycles to answer, and a warp that needs its result waits; the
multiprocessor meanwhile issues instructions from other resident warps that
are ready. With many warps resident, there is almost always one ready, and
the waits are hidden. A full-screen pass has hundreds of thousands of warps
queued for those places, so while one waits, others work.

A tile list at 73% has about 2,000 tiles: about 4,000 warps, which all fit
on the GPU at once. There is no queue behind them, so nothing hides the
waits, and the pass takes as long as one thread's chain of dependent loads:
its records, then (for silhouettes) the outer records, then the spatial
filter's neighbours, each loaded only when the round before says so. The
counters agreed: the dispatches kept 3 to 11 warps resident per
multiprocessor, of 48, and the multiprocessors were busy 10 to 25% of the
time. A pass like that is **latency bound**: it is slow because it waits,
not because there is much to do. Its time hardly depends on how many tiles
it draws, and three dispatches that write the same texture run one after
the other, so three such floors add up instead of overlapping.

So there is one list and one dispatch. The entry still carries the record
types, but every tile runs the same code. Leaving the other type's code out
of a class would save loads and registers, which helps a pass with more
warps than places; it does nothing for a pass whose time is one chain of
loads. With the record mask in place of the direct reads:

| Base clocks | No tiles | One list |
|---|---:|---:|
| Drawing pass, 100% | 0.079 ms | 0.024 ms |
| Drawing pass, 73% | 0.191 ms | 0.069 ms |
| Mask, classify, counter clear and setup | – | 0.023 ms (100%), 0.020 ms (73%) |
| **In-game, 100%** | **0.151 ms** | **0.125 ms (−17%)** |
| **In-game, 73%** | **0.202 ms** | **0.124 ms (−39%)** |

The image did not change: a Python copy of the drawing code, run on the
same frame's records, matched all 20,546 drawn display pixels at 73%.

What was left in the drawing pass, 0.069 ms at 73%, was still that chain of
loads, now run once per listed tile, and at 73% run for each of the three
or so rendering pixels a display pixel overlaps, with every neighbouring
display pixel repeating it for the same rendering pixels. The tile count
was no longer the lever; the length of the chain was. That is what the edge
pass took apart: it selects and filters each rendering pixel's edge once, on
the rendering tiles, and leaves the display pass one load and the band's
area. The
[drawing and performance]({% post_url 2026-10-11-thin-outline-drawing-and-performance %})
post has it and what it saved.

## Drawing in Place

The drawing passes became compute shaders, one 8×8 thread group per listed
tile, and each thread reads and writes its own pixel of the scene colour
through an unordered access view, in place. Writing a new texture instead
would mean copying the 86% of empty tiles into it. And drawing tile quads
with a pixel shader and blending, the usual way to touch part of a target,
is not possible after the upscaler: TSR creates its output as a texture
that shaders read and compute shaders write, not as a render target. Where
the scene colour cannot be loaded and stored this way, the plugin falls back
to the full-screen pixel shader, which also remains for drawing into a
separate output.

## The Overwrite Before the Upscaler, One Quad per Tile

When the outline is drawn before the upscaler, a last pass writes the
foreground's depth and velocity under the silhouette pixels, so that the
upscaler moves the outline with the object in front (the
[silhouettes]({% post_url 2026-10-11-thin-outline-silhouettes %}) post).
Depth targets cannot be written from compute shaders, so this pass stays a
pixel shader, but it draws one quad per listed tile instead of the whole
screen: an instanced draw started indirectly from the list's length (the
setup pass writes these arguments too), with a vertex shader that reads the
tile list and places each quad. Substrate's tile passes do the same. The
drawing pass writes the foreground's depth and velocity only in the listed
tiles, the only ones the overwrite reads.

At 100% the list held 1,570 of 14,400 rendering tiles (10.9%), and I
expected the pass's 0.037 ms to shrink with the area. It did not:

| Base clocks | No tiles | Tiles |
|---|---:|---:|
| Overwrite, 100% | 0.0366 ms | 0.0317 ms |
| Overwrite, 73% | 0.0293 ms | 0.0284 ms |
| Drawing pass, 100% | 0.0675 ms | 0.0267 ms |
| Drawing pass, 73% | 0.0453 ms | 0.0231 ms |
| **In-game, 100%** | **0.153 ms** | **0.135 ms (−12%)** |
| **In-game, 73%** | **0.098 ms** | **0.091 ms (−7%)** |

The overwrite's pixel warps are idle either way (0.09 active per
multiprocessor and cycle, the multiprocessors at 0.3% of their throughput). The
0.03 ms is not shading; it is what the GPU does around any pass that writes
depth from a pixel shader into the scene depth, whose compression has to be
resolved, and it is paid once per pass whatever is drawn. The quads stay,
since they are not slower, but the pass is a fixed cost as long as it writes
depth that way. Drawing before the upscaler is not the default, so I did not
push this further; its gain is the drawing pass's.

## The Record Pass Skips Empty History

The record pass cannot skip its edge checks, but it can skip part of its
history. A pixel fetches last frame's records from four bilinear taps
around its reprojected position (the
[history reprojection]({% post_url 2026-10-11-thin-outline-history-reprojection %})
post). The record mask is kept with the records for the next frame, and
once the four taps are known, the pass looks up their blocks in last frame's
mask; if none held a record of either type, the fetch returns at once with
empty records, which is exactly what it would have loaded:

{% raw %}
```hlsl
bool HasHistoryRecordsNearTaps(FThinOutlineHistoryTaps Taps)
{
	if (bHistoryMaskValid == 0)
	{
		return true;
	}

	bool bFound = false;
	UNROLL
	for (uint Index = 0; Index < 4; ++Index)
	{
		if (Taps.Weights[Index] > 0.0)
		{
			const int2 Pixel = Taps.Pixels[Index];
			const uint Mask = HistoryRecordMask.Load(int3(Pixel / THIN_OUTLINE_TILE_SIZE, 0));
			const int2 Block = (Pixel % THIN_OUTLINE_TILE_SIZE) / THIN_OUTLINE_MASK_BLOCK_SIZE;
			const uint Bit = 1u << uint(Block.y * THIN_OUTLINE_MASK_BLOCKS_PER_GROUP + Block.x);
			bFound = bFound || (Mask & (Bit | (Bit << THIN_OUTLINE_MASK_SILHOUETTE_SHIFT))) != 0;
		}
	}
	return bFound;
}
```
{% endraw %}

No margin for motion is needed, because the lookup is made where the taps
are, after the velocity has moved them. Without last frame's mask (the
first frame, or after the history was reset), it fetches as before.

| Base clocks, drawn after the upscaler | Before the skip | With the skip |
|---|---:|---:|
| Record pass, 100% | 0.1107 ms | 0.1053 ms (−5%) |
| Record pass, 73% | 0.0636 ms | 0.0618 ms (−3%) |
| **In-game, 100%** | 0.1249 ms | **0.1210 ms** |
| **In-game, 73%** | 0.1240 ms | **0.1213 ms** |

Less than I hoped from an earlier probe that removed the history fetch
altogether, which put it at a third of the pass. The loads of an empty
neighbourhood were mostly cache hits, and the pass hid their latency behind
its edge checks anyway. The edge checks, which read the G-buffer at every
pixel every frame, are the record pass, and no tile list can skip them. The
skip compiled in without raising the pass's register count, so it cost
nothing in resident warps (about 14 per multiprocessor either way; the
[drawing and performance]({% post_url 2026-10-11-thin-outline-drawing-and-performance %})
post explains why that number matters for the record pass).

## Where It Stands

The figure shows the passes of a frame as they run now:

![A diagram of the passes in three rows. The first row, before the upscaler at rendering resolution in both drawing modes: the record pass on every rendering pixel, then the record mask with one integer per square, then classify and setup over rendering tiles writing a list and indirect arguments, then the edge pass, started indirectly on the listed tiles. A dashed arrow from the record mask loops back to the record pass, labelled next frame: skip the history where its taps' blocks were empty. The second row, drawing before the upscaler: the composite, indirect on the same list and in place, and the depth and velocity overwrite, an indirect draw of a quad per tile, both fed from the edge pass. The third row, the default, drawing after the upscaler at display resolution: the temporal upscaler, then classify and setup over display tiles, fed by the record mask, then the composite, indirect on the listed display tiles and in place, which reads the edge pass's lines, one per rendering pixel. Blue boxes run only on the listed tiles, started indirectly from the count on the GPU](/assets/images/thin-outline/tile-passes.svg)

In-game, the whole plugin, at normal clocks, 1280×720 display, static view,
before the edge pass:

| Drawing mode | Before the tiles | With the tiles |
|---|---:|---:|
| After the upscaler, 100% | 0.151 ms | 0.121 ms (−20%) |
| After the upscaler, 73% | 0.202 ms | 0.121 ms (−40%) |
| Before the upscaler, 100% | 0.153 ms | 0.135 ms (−12%) |
| Before the upscaler, 73% | 0.098 ms | 0.091 ms (−7%) |

At base clocks, drawing after the upscaler at 73%, that was the record pass
0.062 ms, the mask 0.009 ms, the list passes 0.011 ms, and the drawing pass
0.069 ms. The edge pass then cut the plugin by another 32% at 73% after the
upscaler; the
[drawing and performance]({% post_url 2026-10-11-thin-outline-drawing-and-performance %})
post has the current costs with every part of the algorithm switched on.

What the tiles cannot save, and what is still open:

- **The edge checks**, more than half of the record pass, which run on every
  pixel every frame.
- **The mask as a pass of its own** (0.009–0.012 ms). The record pass could
  OR the bits together at its end for free, unless doing so raises its
  register count by a step, which would cost more than it saves.
- **The small passes' fixed cost**: clearing the counter, classifying and
  the setup take about 0.011 ms together, a cost that does not shrink with
  the screen. Folding them into fewer passes would save about half of it,
  and at low resolutions it matters more.
