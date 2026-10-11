---
layout: post
title: "Thin Outlines From the G-Buffer Under TAA: Drawing After the Upscaler"
date: 2026-10-11 15:45:00 +0900
categories: [rendering, npr]
hidden: true
---

The records live at rendering resolution, before the temporal upscaler,
but nothing forces the outline to be drawn there. This post covers where in
the frame the outline is painted: before the upscaler, where it goes
through TSR like any other pixel, or after it, at display resolution, where
each display pixel takes the exact coverage of the band. It explains why
drawing after the upscaler became the default, what it costs, the edge pass
that made it cheap, and where the remaining time goes. It is the last part
of a [series]({% post_url 2026-10-11-thin-outline %}) and assumes the
[core idea]({% post_url 2026-10-11-thin-outline-core-idea %}) post.

## Two Places to Draw

The record pass has to run before the upscaler: it needs this frame's
jittered G-buffer at rendering resolution, and the jitter is what it
samples. In Unreal it runs at the hook before depth of field, after the
opaque surfaces, the fog and the translucency have been rendered. What
happens to the outline after that depends on where it is painted:

![A vertical list of the stages of a frame, top to bottom: opaque surfaces and lighting (the jittered G-buffer); fog and sky haze, applied to scene colour; translucency, with "After DOF" materials rendered into a texture of their own; the Before DOF hook, outlined in blue, where the record pass and the edge pass run; depth of field, which composites the "After DOF" translucency; TSR, the temporal upscaler; motion blur; the After motion blur hook, outlined in purple; bloom and tonemapping. A bracket on the left marks everything down to TSR as rendering resolution, jittered, and everything after as display resolution. A blue note at the Before DOF hook says that when drawing before the upscaler the composite and the depth and velocity overwrite also run there, and the outline then goes through depth of field and TSR and sits under the "After DOF" translucency. A purple note at the After motion blur hook says that drawing after the upscaler, the default, runs the composite there at display resolution from the edge pass's lines, drawn over everything composited so far, and stays sharp](/assets/images/thin-outline/draw-hooks.svg)

- **Before the upscaler**, the outline is painted into the scene colour at
  rendering resolution, and TSR accumulates it with everything else. Each
  frame a pixel passes the band's coverage of the display pixel under its
  jittered sample, and the upscaler's average over frames is the display
  pixel's coverage (the core idea post's "Coverage-Based Alpha"). The
  upscaler moves the outline by the motion vectors, so silhouette pixels
  need the foreground's depth and velocity written under them (the
  [silhouettes]({% post_url 2026-10-11-thin-outline-silhouettes %}) post).
- **After the upscaler**, the records are read by a pass at display
  resolution, after motion blur and before bloom and tonemapping. Each
  display pixel computes its coverage directly, every frame, and the
  upscaler never sees the outline. Nothing needs to be overwritten.

Both hooks are before bloom and tonemapping, so the outline's colour goes
through the same colour stages either way, and a comparison between them
shows only what the upscaler does.

## What Drawing After the Upscaler Showed

Some outlines came out dashed in my first drawing, and some faded or
blinked. I had blamed the records, the seams between neighbouring pixels'
lines, and I had a spatial filter half planned for it. Before writing it, I
wanted to know how much of it was the upscaler, so I added the second
drawing mode as a switch, to compare the two on the same records. Drawn
after the upscaler, the fading and the dashed lines were gone. They had
come from TSR's handling of a thin outline that changes with the jitter,
not from the records. I did not take apart which of TSR's heuristics did
it: drawing after it removed the question, and it became the default.

It also showed something TSR had been hiding. Below 100% screen percentage
some lines came out as a two-row sawtooth: every rendering pixel along a
static edge holds the same jitter samples, so all of them fit the same
slope, with the same error, and the line leans the same way in each pixel
and jumps back at the next. Before the upscaler, each frame reported the
display pixel under the sample, the top half of the rendering pixel on some
frames and the bottom half on others, and TSR averaged the lean away. After
it, nothing averages it. That is what the
[spatial filters]({% post_url 2026-10-11-thin-outline-spatial-filters %})
post straightens.

## How a Display Pixel Is Drawn

The records' lines are in the coordinates of the rendering pixels without
the jitter, since the jitter only says where each frame's sample lay. A
display pixel covers a known rectangle of that space: with $s$ rendering
pixels per display pixel ($s = 0.5$ at 50% screen percentage), display
pixel $d$ covers $[d\,s,\ (d + 1)\,s]$ along each axis. Each rendering
pixel $R$ it overlaps contributes the part of its own band inside the
overlap:

$$
\alpha_D = \sum_{R} s_R \cdot \frac{\lvert\, \text{band}_R \cap D \cap R \,\rvert}{\lvert D \rvert},
$$

with $s_R$ the strength of $R$'s edge. The rendering pixels draw separate
parts of the display pixel, so their alphas add up. This is the same rule
as before the upscaler, "each rendering pixel draws the part of the band
that crosses it", with the display pixel's own rectangle instead of the one
under this frame's sample, and it is the sum the core idea post showed the
upscaler converging to, now computed in one frame. At 100% and 50% a display
pixel lies inside one rendering pixel; at 73% it can straddle four:

![Four square rendering pixels R(0, 0), R(1, 0), R(0, 1) and R(1, 1) in a 2 by 2 grid, with a black edge line running down and to the right through them, x equals 0.70 plus 0.40 y in rendering pixels, and a light grey band half a display pixel thick around it. A purple square, display pixel D, 0.73 rendering pixels on a side, straddles the grid's centre. Inside D the band is divided by the rendering pixels' borders into four coloured pieces, each in its dashed overlap rectangle D intersected with R: green in R(0, 0), 0.084 of an overlap of 0.171; orange in R(1, 0), 0.093 of 0.158; blue in R(0, 1), 0.012 of 0.107; pink in R(1, 1), 0.094 of 0.098. The legend gives the display pixel's area as 0.534 and the alpha as the four band areas summed and divided by it, 0.283 over 0.534, 0.53, at strength 1, and notes that since every R draws the same line here the sum is the band's exact share of D](/assets/images/thin-outline/display-pixel-overlap.svg)

When the four rendering pixels draw the same line, as neighbours along one
edge should, the sum is the exact coverage of the display pixel. When they
disagree, each one still draws only inside its own part of the display
pixel, so a seam shows as a small step at a rendering pixel's border, not
as a doubled line. The pass, shortened:

{% raw %}
```hlsl
// The display pixel in rendering pixels from the view's origin, and the rendering pixels it overlaps.
const float2 DisplayRectMin = float2(OutputViewportPos) * RenderPixelsPerDisplayPixel;
const float2 DisplayRectMax = (float2(OutputViewportPos) + 1.0) * RenderPixelsPerDisplayPixel;
const float DisplayRectArea = RenderPixelsPerDisplayPixel.x * RenderPixelsPerDisplayPixel.y;
const int2 FirstPixel = max(int2(floor(DisplayRectMin + THIN_OUTLINE_MIN_OVERLAP)), 0);
const int2 LastPixel = min(int2(ceil(DisplayRectMax - THIN_OUTLINE_MIN_OVERLAP)) - 1, ViewSize - 1);

for (int Y = FirstPixel.y; Y <= LastPixel.y; ++Y)
{
	for (int X = FirstPixel.x; X <= LastPixel.x; ++X)
	{
		const int2 ViewportPos = int2(X, Y);
		const FThinOutlineDrawnEdge Edge = UnpackDrawnEdge(DrawnEdge.Load(int3(ViewportPos, 0)));
		if (Edge.bValid && !Edge.bSuppressed)
		{
			// The overlap, in pixel-local coordinates of the rendering pixel.
			const float2 RectMin = max(DisplayRectMin - float2(ViewportPos), 0.0);
			const float2 RectMax = min(DisplayRectMax - float2(ViewportPos), 1.0);
			const float BandArea = max(GetCandidateBandArea(Edge.Candidate, Edge.bVertical, RectMin, RectMax), 0.0);
			const float Alpha = BandArea / DisplayRectArea * max(Edge.Candidate.Strength, 0.0);
			HorizontalAlpha += Edge.bVertical ? 0.0 : Alpha;
			VerticalAlpha += Edge.bVertical ? Alpha : 0.0;
			SilhouetteAlpha += Edge.bSilhouette ? Alpha : 0.0;
		}
	}
}
```
{% endraw %}

`GetCandidateBandArea` is the closed-form band area of the core idea post,
on the axis the edge belongs to. Overlaps thinner than a thousandth of a
rendering pixel are rounding of the pixel ratio and are skipped. The
display pixel is then blended toward the crease colour by the creases'
share of the alpha and toward the silhouette colour by the silhouettes',
with the total clamped to 1. The thickness is set in display pixels for
both types and converted to rendering pixels by the screen percentage, so
an outline half a display pixel thick is half a display pixel thick at any
screen percentage.

## What Each Mode Trades

| | Before the upscaler | After the upscaler |
|---|---|---|
| Upscaler history | the outline is accumulated, filtered and moved by TSR | not involved: each frame draws its own outline |
| Depth and velocity overwrite | needed under silhouette pixels | none; scene depth and velocity stay untouched |
| Depth of field, motion blur | drawn over the outline | the outline is drawn over them and stays sharp |
| "After DOF" translucency (the material default) | covers the outline | the outline is drawn over it |
| Thickness | in display pixels, through the upscaler's filter | exactly in display pixels |
| Pixels the composite runs on | rendering pixels | display pixels: 4× as many at 50% |

Neither mode puts the outline under the fog. Fog is applied to the scene
colour before either hook, and the engine has no hook between lighting and
fog. Making the outline sit on
its surface would mean fogging the outline colour itself, the way
translucent materials are fogged, at the crease's depth or the
silhouette's foreground depth, and, after the upscaler, painting it under
the translucency texture rather than over the composited image. I have not
done either; the main post lists it among the limitations.

Drawing after the upscaler also costs more below 100%, since it runs on
more pixels. When I first added it (1280×720 display, in-game, medians of
600-frame profiles):

| Screen % | Before the upscaler | After the upscaler |
|---|---:|---:|
| 100% | 0.1347 ms | 0.1337 ms (−1%) |
| 50% | 0.0529 ms | 0.0747 ms (+41%) |

At 100% the composite ran on as many pixels as before and the overwrite was
gone. At 50% it ran on four times as many. Below 100% but above 50% it was
worse than that, since a display pixel straddles up to four rendering pixels
and evaluated each one's edge, and its neighbours evaluated the same
rendering pixels again. The
[tile classification]({% post_url 2026-10-11-thin-outline-tile-classification %})
post removed the pixels with nothing to draw, and the edge pass removed the
repetition.

## The Edge Pass

Choosing a rendering pixel's edge is most of the drawing work: pooling its
records with its neighbours', fitting both axes, choosing between them, the
[spatial filter]({% post_url 2026-10-11-thin-outline-spatial-filters %})
across the neighbouring rows, and the
[dense and isolated edge suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %}).
At first the composite after the upscaler did all of it inside the display
pass, for every rendering pixel a display pixel overlapped: at 73%, about
three per display pixel, and every rendering pixel's edge chosen again by
each display pixel it touched. Measured at base clocks, that pass was a
chain of dependent loads (the records, then the outer records, then the
filter's neighbours), and after the tile classification its time was that
chain's length, not the number of pixels.

The edge pass takes the choice out. It runs once per rendering pixel, on
the rendering tiles within reach of a record, right after the record pass,
and writes one line per rendering pixel into a four-channel float texture:
the intercept, the slope, the strength, and flags for valid, silhouette,
vertical axis, background side, and suppressed. The composites, before or
after the upscaler, then only compute the band's coverage from it: a
display pixel loads one texel per rendering pixel it overlaps.

The output did not change: my Python copy of the shaders, which had not
been changed, still matched the image pixel for pixel. The cost, in-game,
1280×720 display, medians of in-game profiles:

| Configuration | Without the edge pass | With it |
|---|---:|---:|
| After the upscaler, 73% | 0.1374 ms | 0.0937 ms (−32%) |
| After the upscaler, 73%, dense edge suppression on | 0.1466 ms | 0.0966 ms (−34%) |
| Before the upscaler, 100% | 0.1403 ms | 0.1427 ms (within noise) |

Before the upscaler there was nothing repeated to remove, one composite
pixel per rendering pixel, so the edge pass only moved the work into a pass
of its own.

## What It Costs

In-game, the whole algorithm at its default settings (presence, spike
filter, spatial filter, dense and isolated suppression and the velocity test
on), 1280×720 display on an RTX 5080, the static view of my test scene,
medians of in-game profiles at normal clocks:

| Drawing mode | Outline cost | Whole frame |
|---|---:|---:|
| After the upscaler, 73% | 0.106 ms | 2.87 ms |
| Before the upscaler, 100% | 0.160 ms | 3.46 ms |

The mode before the upscaler at 100% costs more mostly because it has
almost twice the rendering pixels (921,600 against 491,810 at 73%), and the
record pass runs on every one of them. It also carries the overwrite, which
the tile classification post found to be a fixed cost of about 0.03 ms
(base clocks) for writing depth from a pixel shader, whatever it draws.

I have not traced the passes one by one since the edge pass. Before it, at
base clocks and 73% after the upscaler, the record pass took 0.062 ms, the
tile lists 0.020 ms, and the composite 0.069 ms; the edge pass took most of
the composite's share away. The record pass is now the largest part, and the
one no tile list can shrink.

### The Record Pass Waits on Its Loads

The record pass reads the G-buffer around every rendering pixel, every
frame: nine depths and five normals for the edge checks, then the history
taps. Most of that time is waiting. A GPU runs threads in warps of 32, and
each multiprocessor keeps up to 48 warps resident; while one warp waits for
a texture load, others issue instructions. With few resident warps, all of
them end up waiting at once, and the multiprocessor idles. The record pass
is like that: its traces showed the schedulers issuing on about half the
cycles, with the memory system far from busy.

The number of resident warps is set by registers. A multiprocessor has
65,536 registers, shared by its resident warps, and each thread of a shader
reserves as many as the program holds at its heaviest point:

| Registers per thread | Warps that fit (of 48) |
|---:|---:|
| 64 | 32 |
| 80 | 24 |
| 96 | 20 |
| 128 | 16 |

Two consequences shaped the record pass. The heaviest point counts even if
it rarely runs: a branch that one pixel in fifty takes, holding more values
at once, reserves those registers for every warp. And values that stay
alive cost their registers the whole time: the silhouette history is
fetched before the crease history, so the crease records do not yet exist
while the heavier silhouette fetch holds its taps and tests.

What helped more than the occupancy was removing repeated work. Probes that
deleted one part of the pass at a time (wrong images, timing only) put the
edge checks at more than half of it, 0.064 of 0.113 ms at 100% and base
clocks, and the history fetch at about 0.033 ms; the resident warps hardly
moved whichever part was removed, so neither part alone set the register
count. In the edge checks, the depth-edge measure toward each of the four
neighbours was computed up to three times, by the silhouette check, by the
crease check's same-surface test, and by the opposite direction's test:
twelve measures where only four differ. Computing each once, and rebuilding
the nine world positions from the linear depth (a 4×3 transform without a
division) instead of a full 4×4 transform each, took the pass from 0.1142
to 0.0989 ms at the same 17 resident warps per multiprocessor (base clocks,
100%, static camera).

Additions since then raised the pass one register step, to about 111 registers per thread
and 14 resident warps. Getting back below that step, which would be worth
about 15% of a latency-bound pass, is the main open item of the
implementation, together with the edge checks themselves, which every
pixel runs every frame.

## Using It

The plugin, ThinOutline, is at
<https://github.com/donguklim/thin-outline-plugin>, for a source build of
Unreal Engine 5.8 with TSR. Its settings are under Project Settings,
Plugins, Thin Outline:

- **General:** enabling the outlines, and drawing after the upscaler (on by
  default) or before it.
- **Silhouette** and **Crease:** whether to draw each type, its colour, its
  thickness in display pixels, and the detection thresholds and scales of
  each type's check.
- The other categories (presence, spike filter, velocity test, estimator)
  hold the parameters of the steps the other posts describe.

Drawing after the upscaler is the right choice unless the outline has to go
under depth of field, motion blur or "After DOF" translucency; then drawing
before it, with the overwrite, is the one that does.

## Summary

| Part | What it does | Where |
|---|---|---|
| Record pass | samples the jittered G-buffer, updates the records | rendering resolution, before depth of field, every pixel |
| Edge pass | chooses, filters and suppresses each rendering pixel's edge once | rendering resolution, listed tiles |
| Composite after the upscaler (default) | sums each display pixel's band coverage from the lines it overlaps | display resolution, after motion blur, listed tiles |
| Composite before the upscaler | passes the coverage of the display pixel under the sample, for TSR to average | rendering resolution, listed tiles |
| Overwrite (before the upscaler only) | writes the foreground's depth and velocity under silhouettes | listed tiles, a fixed cost |

The records are where the algorithm is; drawing is where the upscaler can
either help or get in the way. For a thin outline that changes with the
jitter, keeping the upscaler out of it was what made the outline steady.
