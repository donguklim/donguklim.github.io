---
layout: post
title: "Thin Outlines From the G-Buffer Under TAA: Drawing After the Upscaler"
date: 2026-10-11 15:45:00 +0900
categories: [rendering, npr]
hidden: true
---

The records live at rendering resolution, before the temporal upscaler, but nothing forces the outline to be drawn
there. This post covers where in the frame the outline is painted. Before the upscaler, it goes through TSR like any
other pixel. After it, at display resolution, each display pixel takes the exact coverage of the band. The post
explains what drawing after the upscaler showed, what it costs, the edge pass that made it cheap, and where the
remaining time goes. It is the last part of a [series]({% post_url 2026-10-11-thin-outline %}) and assumes the
[core idea]({% post_url 2026-10-11-thin-outline-core-idea %}) post.

## Two Places to Draw

The record pass has to run before the upscaler. It needs this frame's jittered G-buffer at rendering resolution, and
the jitter is what it samples. In Unreal, it runs at the hook before depth of field, after the opaque surfaces, the
fog and the translucency have been rendered. What happens to the outline after that depends on where it is painted:

![A vertical list of the stages of a frame, top to bottom: opaque surfaces and lighting (the jittered G-buffer); fog and sky haze, applied to scene colour; translucency, with "After DOF" materials rendered into a texture of their own; the Before DOF hook, outlined in blue, where the record pass and the edge pass run; depth of field, which composites the "After DOF" translucency; TSR, the temporal upscaler; motion blur; the After motion blur hook, outlined in purple; bloom and tonemapping. A bracket on the left marks everything down to TSR as rendering resolution, jittered, and everything after as display resolution. A blue note at the Before DOF hook says that when drawing before the upscaler, the default, the composite and the depth and velocity overwrite also run there, and the outline then goes through depth of field and TSR and sits under the "After DOF" translucency. A purple note at the After motion blur hook says that drawing after the upscaler runs the composite there at display resolution from the edge pass's lines, drawn over everything composited so far, and stays sharp](/assets/images/thin-outline/draw-hooks.svg)

- **Before the upscaler**, the outline is painted into the scene colour at rendering resolution, and TSR accumulates
  it with everything else. Each frame, a pixel passes the band's coverage of the display pixel under its jittered
  sample. The upscaler's average over frames is then the display pixel's coverage (the core idea post's
  "Coverage-Based Alpha"). The upscaler moves the outline by the motion vectors, so silhouette pixels need the
  foreground's depth and velocity written under them (the
  [silhouettes]({% post_url 2026-10-11-thin-outline-silhouettes %}) post).
- **After the upscaler**, a pass at display resolution reads the records, after motion blur and before bloom and
  tonemapping. Each display pixel computes its coverage directly, every frame, and the upscaler never sees the
  outline. Nothing needs to be overwritten.

Both hooks are before bloom and tonemapping. So the outline's colour goes through the same colour stages either way,
and a comparison between them shows only what the upscaler does.

## What Drawing After the Upscaler Showed

### The Fading Came From TSR

In my first drawing, some outlines came out dashed, and some faded or blinked. I had blamed the records: the seams
between neighbouring pixels' lines. I had a spatial filter half planned for them. Before writing it, I wanted to know
how much of it was the upscaler. So I added the second drawing mode as a switch, to compare the two on the same
records.

Drawn after the upscaler, the fading and the dashed lines were gone. They had come from TSR's handling of a thin
outline that changes with the jitter, not from the records. I did not take apart which of TSR's heuristics did it:
drawing after it removed the question.

### A Sawtooth That TSR Had Hidden

Below 100% screen percentage, some lines drawn after the upscaler came out as a two-row sawtooth. Every rendering
pixel along a static edge holds the same jitter samples. So all of them fit the same slope, with the same error, and
the line leans the same way in each pixel and jumps back at the next.

Before the upscaler, each frame reported the display pixel under the sample: the top half of the rendering pixel on
some frames, the bottom half on others. TSR averaged the lean away. After the upscaler, nothing averages it. That is
what the [spatial filters]({% post_url 2026-10-11-thin-outline-spatial-filters %}) post straightens.

## How a Display Pixel Is Drawn

The records' lines are in the coordinates of the rendering pixels without the jitter, since the jitter only says
where each frame's sample lay. A display pixel covers a known rectangle of that space. Let $s$ be the number of
rendering pixels per display pixel ($s = 0.5$ at 50% screen percentage). Display pixel $d$ then covers
$[d\,s,\ (d + 1)\,s]$ along each axis. Each rendering pixel $R$ it overlaps contributes the part of its own band
inside the overlap:

$$
\alpha_D = \sum_{R} s_R \cdot \frac{\lvert\, \text{band}_R \cap D \cap R \,\rvert}{\lvert D \rvert},
$$

with $s_R$ the strength of $R$'s edge. The rendering pixels draw separate parts of the display pixel, so their alphas
add up.

This is the same rule as before the upscaler: "each rendering pixel draws the part of the band that crosses it". Only
the rectangle differs: the display pixel's own, instead of the one under this frame's sample. It is also the sum the
core idea post showed the upscaler converging to, now computed in one frame.

At 100% and 50%, a display pixel lies inside one rendering pixel. At 73%, it can straddle four:

![Four square rendering pixels R(0, 0), R(1, 0), R(0, 1) and R(1, 1) in a 2 by 2 grid, with a black edge line running down and to the right through them, x equals 0.70 plus 0.40 y in rendering pixels, and a light grey band half a display pixel thick around it. A purple square, display pixel D, 0.73 rendering pixels on a side, straddles the grid's centre. Inside D the band is divided by the rendering pixels' borders into four coloured pieces, each in its dashed overlap rectangle D intersected with R: green in R(0, 0), 0.084 of an overlap of 0.171; orange in R(1, 0), 0.093 of 0.158; blue in R(0, 1), 0.012 of 0.107; pink in R(1, 1), 0.094 of 0.098. The legend gives the display pixel's area as 0.534 and the alpha as the four band areas summed and divided by it, 0.283 over 0.534, 0.53, at strength 1, and notes that since every R draws the same line here the sum is the band's exact share of D](/assets/images/thin-outline/display-pixel-overlap.svg)

When the four rendering pixels draw the same line, as neighbours along one edge should, the sum is the exact coverage
of the display pixel. When they disagree, each one still draws only inside its own part of the display pixel. So a
seam shows as a small step at a rendering pixel's border, not as a doubled line.

### The Pass

Here is the pass, shortened:

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

- `GetCandidateBandArea` is the closed-form band area of the core idea post, on the axis the edge belongs to.
- Overlaps thinner than a thousandth of a rendering pixel are rounding of the pixel ratio and are skipped.
- The display pixel is then blended toward the crease colour by the creases' share of the alpha, and toward the
  silhouette colour by the silhouettes'. The total is clamped to 1.
- The thickness is set in display pixels for both types and converted to rendering pixels by the screen percentage.
  So an outline half a display pixel thick is half a display pixel thick at any screen percentage.

### Beads on a Diagonal, and the Coverage Curve

#### The Beads

Exact coverage has one effect I had not expected. Take a band one display pixel wide along an 18° edge. It climbs one
row every three columns:

- in one column, it sits centred on a row and covers that pixel almost fully;
- a column or two later, it sits on the border between two rows and covers half of each.

The alpha is spread exactly in both cases. And yet the column on the border looks far lighter, so the line reads as a
chain of beads.

I first suspected the spatial filter. But the
[spatial filters]({% post_url 2026-10-11-thin-outline-spatial-filters %}) post's measurement put the per-pixel lines
on one straight line to 0.04 pixels. And a perfectly straight line painted the same way showed the same beads.

#### Linear Light Is Not Lightness

The cause is where the alpha is applied. The composite lerps the scene colour toward the line colour in linear light,
and the eye does not see linear light. A pixel left with half its luminance reads as about three quarters of its
lightness.

In CIE L\*, the standard lightness scale, a black line on a white surface removes:

- 92 units when it fills one pixel;
- 22 from each of two half-covered pixels, 44 in all, when it is split over them.

So the single-pixel column looks twice as dark as its neighbour.

#### Bending the Alpha

The remedy I chose keeps the band and the pass as they are. It bends the alpha before the lerp:

$$
\alpha' = 1 - (1 - \alpha)^{\gamma},
$$

with $\gamma = 2.2$, the exponent of the sRGB curve, and $\alpha$ the display pixel's coverage times strength. Text
rasterizers do the same, under the names gamma-adjusted coverage and stem darkening.

- A fully covered pixel is unchanged: $\alpha' = 1$.
- A half-covered one gets $\alpha' = 0.78$ instead of 0.5. In linear light, that is the darkness the eye expects of
  half a line.

Take the one-dimensional model of a hard band crossing the pixel grid at every phase. With plain coverage, the
lightness removed per column swings from 44 to 92 units. With the curve, it swings from 86 to 95, while the darkest
pixel of a column stays as dark as before.

In HLSL, it is one function applied to the pixel's summed alpha. The silhouette and crease shares are scaled alike,
so the colour split stays:

```hlsl
float ApplyCoverageGamma(float Alpha)
{
	return 1.0 - pow(max(1.0 - Alpha, 0.0), CoverageGamma);
}
```

#### In Screenshots

Here are two screenshots of the same view at 50%, zoomed four times, with the curve on at the left and off at the
right. With the curve, the slanted edge of the block and the top of the wall are solid lines. Without it, they break
into dashes:

![Two crops of the same green block seen from the side, each 280 by 80 display pixels zoomed four times and separated by a magenta bar. On the left, with the coverage curve, the dark outline along the block's slanted top-left edge and along the top of the wall behind it are continuous, evenly dark lines; on the right, without it, the same edges are chains of short dark dashes separated by lighter gaps where the band straddles two pixel rows](/assets/images/thin-outline/coverage-gamma-right.png)

![Two crops of a grey wall against a purple sky, each 300 by 80 display pixels zoomed four times and separated by a magenta bar. The wall's top edge runs almost horizontally with a slope of 0.06; on the left, with the coverage curve, it is a single dark line, on the right, without it, the line fades and breaks every sixteen pixels where it moves from one row to the next](/assets/images/thin-outline/coverage-gamma-left.png)

Measured along the wall's top in those screenshots, the lightness removed per column went from 71 ± 10 units to
89 ± 9: a darker line with a smaller swing.

#### What the Curve Does Not Do

The curve does not make the pixels equal. Zoomed in, the fully covered pixel is still black beside two dark-grey
ones, so a pixel-level inspection still shows a pattern. The curve evens the darkness per column, which is what the
eye sums at a viewing distance. It also assumes the usual display response over the mid-tones.

#### Two Other Remedies

- **Blending after the tonemapper**, in the display's own encoding, would be the exact way: it makes the lerp
  perceptual for free. But I did not want the outline to depend on another post-processing step, or on which
  tonemapper a project uses. The curve gets most of the way there inside the existing pass.
- **A softer band profile**, which ramps off over a display pixel, also reduces the beads. But it does so by cutting
  the dark columns down rather than filling the light ones in, at the price of a lighter line.

## What Each Mode Trades

| | Before the upscaler | After the upscaler |
|---|---|---|
| Upscaler history | the outline is accumulated, filtered and moved by TSR | not involved: each frame draws its own outline |
| Depth and velocity overwrite | needed under silhouette pixels | none; scene depth and velocity stay untouched |
| Depth of field, motion blur | drawn over the outline | the outline is drawn over them and stays sharp |
| "After DOF" translucency (the material default) | covers the outline | the outline is drawn over it |
| Thickness | in display pixels, through the upscaler's filter | exactly in display pixels |
| Pixels the composite runs on | rendering pixels | display pixels: 4× as many at 50% |

### The Outline Is Not Fogged

Neither mode puts the outline under the fog. Fog is applied to the scene colour before either hook, and the engine
has no hook between lighting and fog. Making the outline sit on its surface would mean two things:

1. fogging the outline colour itself, the way translucent materials are fogged, at the crease's depth or the
   silhouette's foreground depth;
2. after the upscaler, painting it under the translucency texture rather than over the composited image.

I have not done either. The main post lists it among the limitations.

### More Pixels Below 100%

Drawing after the upscaler also costs more below 100%, since it runs on more pixels. When I first added it
(1280×720 display, in-game, medians of 600-frame profiles):

| Screen % | Before the upscaler | After the upscaler |
|---|---:|---:|
| 100% | 0.1347 ms | 0.1337 ms (−1%) |
| 50% | 0.0529 ms | 0.0747 ms (+41%) |

- At 100%, the composite ran on as many pixels as before, and the overwrite was gone.
- At 50%, it ran on four times as many.
- Below 100% but above 50%, it was worse than that. A display pixel straddles up to four rendering pixels, and it
  evaluated each one's edge. Its neighbours evaluated the same rendering pixels again.

The [tile classification]({% post_url 2026-10-11-thin-outline-tile-classification %}) post removed the pixels with
nothing to draw, and the [edge pass](#the-edge-pass) removed the repetition.

## The Edge Pass

### The Repeated Choice

Choosing a rendering pixel's edge is most of the drawing work:

- pooling its records with its neighbours';
- fitting both axes and choosing between them;
- the [spatial filter]({% post_url 2026-10-11-thin-outline-spatial-filters %}) across the neighbouring rows;
- the [dense and isolated edge suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %}).

At first, the composite after the upscaler did all of it inside the display pass, for every rendering pixel a display
pixel overlapped. At 73%, that is about three per display pixel. So every rendering pixel's edge was chosen again by
each display pixel it touched.

Measured at base clocks, that pass was a chain of dependent loads: the records, then the outer records, then the
filter's neighbours. After the tile classification, its time was that chain's length, not the number of pixels.

### What the Edge Pass Writes

The edge pass takes the choice out of the composite. It runs once per rendering pixel, right after the record pass,
on the rendering tiles within reach of a record. It writes one line per rendering pixel into a four-channel float
texture:

- the intercept;
- the slope;
- the strength;
- flags for valid, silhouette, vertical axis, background side, and suppressed.

The composites, before or after the upscaler, then only compute the band's coverage from it. A display pixel loads
one texel per rendering pixel it overlaps.

### Same Image, Less Time

The output did not change: my Python copy of the shaders, which I had not changed, still matched the image pixel for
pixel. The cost, in-game, at 1280×720 display, from medians of in-game profiles:

| Configuration | Without the edge pass | With it |
|---|---:|---:|
| After the upscaler, 73% | 0.1374 ms | 0.0937 ms (−32%) |
| After the upscaler, 73%, dense edge suppression on | 0.1466 ms | 0.0966 ms (−34%) |
| Before the upscaler, 100% | 0.1403 ms | 0.1427 ms (within noise) |

Before the upscaler, there was nothing repeated to remove: one composite pixel per rendering pixel. So the edge pass
only moved the work into a pass of its own.

## What It Costs

These are the costs of the whole algorithm in-game:

- at its default settings (presence, spike filter, spatial filter, dense and isolated suppression and the velocity
  test on);
- at 1280×720 display on an RTX 5080;
- in the static view of my test scene;
- as medians of in-game profiles at normal clocks.

| Drawing mode | Outline cost | Whole frame |
|---|---:|---:|
| After the upscaler, 73% | 0.106 ms | 2.87 ms |
| Before the upscaler, 100% | 0.160 ms | 3.46 ms |

The mode before the upscaler at 100% costs more mostly because it has almost twice the rendering pixels (921,600
against 491,810 at 73%). The record pass runs on every one of them. It also carries the overwrite: the tile
classification post found that writing depth from a pixel shader has a fixed cost of about 0.03 ms (base clocks),
whatever it draws.

### Where the Time Goes

I have not traced the passes one by one since the edge pass. Before it, at base clocks and 73% after the upscaler:

| Pass | Time |
|---|---:|
| Record pass | 0.062 ms |
| Tile lists | 0.020 ms |
| Composite | 0.069 ms |

The edge pass took most of the composite's share away. The record pass is now the largest part, and the one no tile
list can shrink.

### The Record Pass Waits on Its Loads

The record pass reads the G-buffer around every rendering pixel, every frame: nine depths and five normals for the
edge checks, then the history taps. Most of that time is waiting.

#### Resident Warps Hide the Waiting

A GPU runs threads in warps of 32, and each multiprocessor keeps up to 48 warps resident. While one warp waits for a
texture load, others issue instructions. With few resident warps, all of them end up waiting at once, and the
multiprocessor idles.

The record pass is like that. Its traces showed the schedulers issuing on about half the cycles, with the memory
system far from busy.

#### Registers Set the Number of Warps

The number of resident warps is set by registers. A multiprocessor has 65,536 registers, shared by its resident
warps. Each thread of a shader reserves as many as the program holds at its heaviest point:

| Registers per thread | Warps that fit (of 48) |
|---:|---:|
| 64 | 32 |
| 80 | 24 |
| 96 | 20 |
| 128 | 16 |

Two consequences shaped the record pass:

1. **The heaviest point counts even if it rarely runs.** A branch that one pixel in fifty takes, holding more values
   at once, reserves those registers for every warp.
2. **Values that stay alive cost their registers the whole time.** The silhouette history is fetched before the
   crease history, so the crease records do not yet exist while the heavier silhouette fetch holds its taps and
   tests.

#### Removing Repeated Work

What helped more than the occupancy was removing repeated work. Probes deleted one part of the pass at a time (wrong
images, timing only). At 100% and base clocks, they put:

- the edge checks at more than half of the pass, 0.064 of 0.113 ms;
- the history fetch at about 0.033 ms.

The resident warps hardly moved, whichever part was removed. So neither part alone set the register count.

In the edge checks, the depth-edge measure toward each of the four neighbours was computed up to three times:

- by the silhouette check;
- by the crease check's same-surface test;
- by the opposite direction's test.

That made twelve measures where only four differ. Two changes took the pass from 0.1142 to 0.0989 ms, at the same
17 resident warps per multiprocessor (base clocks, 100%, static camera):

- computing each measure once;
- rebuilding the nine world positions from the linear depth (a 4×3 transform without a division), instead of a full
  4×4 transform each.

#### What Is Still Open

Additions since then raised the pass one register step, to about 111 registers per thread and 14 resident warps.
Getting back below that step would be worth about 15% of a latency-bound pass. It is the main open item of the
implementation, together with the edge checks themselves, which every pixel runs every frame.

## Using It

The plugin, ThinOutline, is at
<https://github.com/donguklim/thin-outline-plugin>, for a source build of
Unreal Engine 5.8 with TSR. Its settings are under Project Settings, Plugins, Thin Outline:

- **General:** enabling the outlines, drawing before the upscaler (the default) or after it, and the coverage
  curve's exponent (2.2; 1 is plain coverage).
- **Silhouette** and **Crease:** whether to draw each type, its colour, its thickness in display pixels, and the
  detection thresholds and scales of each type's check.
- The other categories (presence, spike filter, velocity test, estimator) hold the parameters of the steps the other
  posts describe.

### Choosing the Drawing Mode

Drawing before the upscaler, the default, treats the outline like the rest of the scene. It goes under depth of
field, motion blur and "After DOF" translucency, and the upscaler accumulates it with everything else.

TSR makes some thin lines fade. But drawn after the upscaler, with no history behind it, the outline gets noisy in
scenes full of small structures, such as a city or a village with grass. And drawing before DLSS 4.5 gave good
results. TSR is the standard, and I think throwing away everything an upscaler does for the outline is generally not
a good trade.

Drawing after the upscaler keeps the line sharp and exactly as thick as set. But it draws the line over all of those
effects, and it costs more below 100%. It is the one to choose when a crisp line matters more than a steady one.

## Summary

| Part | What it does | Where |
|---|---|---|
| Record pass | samples the jittered G-buffer, updates the records | rendering resolution, before depth of field, every pixel |
| Edge pass | chooses, filters and suppresses each rendering pixel's edge once | rendering resolution, listed tiles |
| Composite after the upscaler | sums each display pixel's band coverage from the lines it overlaps, bends it by the coverage curve | display resolution, after motion blur, listed tiles |
| Composite before the upscaler (default) | passes the coverage of the display pixel under the sample, for TSR to average | rendering resolution, listed tiles |
| Overwrite (before the upscaler only) | writes the foreground's depth and velocity under silhouettes | listed tiles, a fixed cost |

The records are where the algorithm is. Drawing is where the upscaler can either help or get in the way. Keeping the
upscaler out of it removes the fading TSR causes on a thin outline that changes with the jitter. Keeping it in
smooths the noise of a scene full of small structures, which is why the outline is drawn before the upscaler by
default.
