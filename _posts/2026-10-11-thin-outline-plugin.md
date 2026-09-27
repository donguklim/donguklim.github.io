---
layout: post
title: "ThinOutline: TAA-Compatible Toon Outlines Thinner Than a Rendering Pixel"
date: 2026-10-11 16:00:00 +0900
categories: [rendering, npr]
---

ThinOutline is an Unreal Engine plugin that draws toon outlines from G-buffer
data alone, thinner than one rendering pixel, and keeps them crisp under a
temporal upscaler such as TSR. This post introduces the plugin and links to
the posts that explain each part of the algorithm. It replaces my
[post from September]({% post_url 2026-09-06-taa-compatible-toon-outline-gbuffer-contour %}),
which described the first design; the algorithm changed in several places
while I implemented it, and this series describes what the plugin actually
does.

The plugin's code is at
<https://github.com/donguklim/thin-outline-plugin>.

## The Problem

A common way to draw toon outlines is to look for contours in the G-buffer:
compare the depth and normal of neighbouring pixels and flag an edge where
they differ by more than a threshold. This works well until a temporal
upscaler (TAAU, TSR, DLSS, FSR, XeSS) enters the pipeline. The upscaler
renders the scene at a reduced internal resolution, the **screen
percentage**, and reconstructs a full display-resolution image from it over
time. A G-buffer contour detector only has samples at rendering resolution,
so the thinnest outline it can produce is one rendering pixel wide, and after
upscaling that becomes about `1 / screen_percentage` display pixels: two at
50% screen percentage, four at 25%.

![A one-render-pixel-wide G-buffer contour becomes two display pixels wide after upscaling from 50% screen percentage](/assets/images/taa-toon-outline/pixel-thickness-upscale.svg)

Geometry-based outlines, such as an inverted hull, don't have this problem:
the outline is a mesh, rasterized at a slightly different sub-pixel jitter
every frame, and the upscaler accumulates those jittered samples into an
anti-aliased, sub-pixel-accurate edge like any other geometry. A G-buffer
detector's output is a yes-or-no per rendering pixel on a fixed grid, so the
upscaler has nothing finer to accumulate. The outline gets thicker, in
display pixels, exactly when the screen percentage drops.

## What the Plugin Does

ThinOutline turns the detector's yes-or-no answers into a line. Each frame,
the G-buffer is sampled at that frame's jittered sub-pixel position, so each
frame's detection says whether an edge passes between two jittered sample
points. Over the frames of the upscaler's jitter sequence, those answers
trace out the shape of the region where the edge is detectable, and from
that region the plugin reconstructs the edge's line equation inside each
rendering pixel by ordinary least squares, from a handful of running
statistics kept per pixel. The line is then drawn with any thickness you
like, measured in display pixels, with exact coverage per pixel. The outline
no longer depends on the screen percentage.

The plugin draws two kinds of edges: **creases**, where the normal changes
abruptly on a continuous surface, and **silhouettes**, where the depth jumps
between a foreground and a background. Both are reconstructed the same way,
with one record per pixel and per inducer axis (horizontal: the left and
right checks; vertical: the top and bottom checks), and each record is
either a crease or a silhouette record.

## The Pipeline

All of it runs on the GPU, in a few passes per frame:

1. **The record pass** (compute, at rendering resolution, before the
   upscaler). For every rendering pixel it fetches last frame's records at
   the position the motion vectors point to, tests whether they still belong
   to the same surface, decays them, runs this frame's crease and silhouette
   checks on the jittered G-buffer, and adds the samples that fired to the
   records. It writes one record per axis and a coarse mask of where records
   exist.
2. **Tile classification.** The screen is cut into 8×8 tiles, and the later
   passes run only on the tiles that have records within reach, which is a
   small fraction of the screen.
3. **The edge pass** (per rendering pixel, on the listed tiles). It pools
   each pixel's record with its neighbours', fits the edge, picks between the
   horizontal and vertical candidate, runs the spatial filter and the dense
   and isolated edge suppression, and scales the edge's strength by how
   consistently it has been found. The result is one line per rendering
   pixel.
4. **The composite.** By default it runs after the upscaler, at display
   resolution: each display pixel sums the coverage of the outline band of
   every rendering pixel it overlaps, and blends the outline colour in by
   that coverage. In the other mode it runs before the upscaler, at rendering
   resolution, and also overwrites the depth and velocity of silhouette
   pixels so the upscaler moves the outline with the foreground object.

## The Series

Each part of the algorithm has its own post. They can be read independently,
but the core idea is the one the others build on.

- [The core idea]({% post_url 2026-10-11-thin-outline-core-idea %}): why a
  G-buffer check detects an edge in a parallelogram of jitter offsets, how a
  temporal least-squares record per pixel and axis reconstructs the edge
  from that, how records are stored, pooled with their neighbours, fitted and
  trusted, and how each pixel paints the part of the outline band that
  crosses it.
- [Silhouettes]({% post_url 2026-10-11-thin-outline-silhouettes %}): the
  depth-edge measure, which side records a silhouette, why the records are
  kept on the foreground pixels, and the depth and velocity overwrite that
  makes the upscaler move the outline with the object in front.
- [History reprojection]({% post_url 2026-10-11-thin-outline-history-reprojection %}):
  how a record follows its surface from frame to frame, the bilinear taps,
  and the depth, background and velocity tests that decide when a record is
  no longer about the surface under its pixel.
- [Spatial filters]({% post_url 2026-10-11-thin-outline-spatial-filters %}):
  why blending neighbouring lines fails on a static edge and what the plugin
  blends instead, and the dense and isolated edge suppression.
- [Flicker suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %}):
  what still flickers after the upscaler and why, measured over jitter
  cycles, and the presence and spike filter that remove most of it.
- [Drawing and performance]({% post_url 2026-10-11-thin-outline-drawing-and-performance %}):
  coverage of the display pixel, the two drawing modes, tile classification,
  what the record pass costs and why.

## How to Use It

### Enabling It

The plugin was developed against a source build of Unreal Engine 5.8.3, as
an engine plugin under `Engine/Plugins`, and it includes renderer-private
headers; a source build is what I can vouch for. Enable it in the project's
plugin list, and it draws with its defaults. Its settings live under Project
Settings > Plugins > Thin Outline and are mirrored by console variables under
`r.ThinOutline.`, so everything can be changed at runtime from the console.

It needs a temporal upscaler whose jitter is uniform over the pixel: TSR's
Halton sequence is. Without temporal anti-aliasing every sample sits at the
pixel centre, the records degenerate, and nothing is drawn; legacy TAA's
Gaussian jitter biases the reconstruction. I have tested the plugin with TSR
in the engine's third-person template, in the City Sample and in Stack O Bot.

### The Main Settings

| Setting | Default | What it does |
|---|---|---|
| `r.ThinOutline.Enable` | 1 | Turns the plugin on and off. |
| `r.ThinOutline.DrawAfterUpscaler` | 1 | Draws the outline at display resolution after the upscaler; 0 draws it before, into the rendering-resolution scene colour. |
| `r.ThinOutline.Crease`, `r.ThinOutline.Silhouette` | 1, 1 | Draw creases, draw silhouettes. The records are kept either way. |
| Crease Color, Silhouette Color | black | The outline colours, in the settings panel. |
| `r.ThinOutline.Crease.Thickness`, `.Silhouette.Thickness` | 1, 1 | Outline thickness in display pixels. Values below 1 work. |
| `r.ThinOutline.Crease.RidgeThreshold`, `.ValleyThreshold`, `.Scale` | 0.25, 0.25, 4 | How sharp a change of the normal counts as a crease, and how fast the strength rises above the threshold. |
| `r.ThinOutline.Silhouette.Threshold`, `.Scale` | 0.01, 50 | The same for the depth-edge measure of silhouettes. |
| `r.ThinOutline.Estimator.Decay` | 0.04 | The records' decay rate per frame, about a 25-frame memory. Slower is steadier, faster follows motion sooner. |
| `r.ThinOutline.Presence`, `.Presence.*` | 1 | Fades out edges that stop being found instead of dropping them, and ramps new edges in. |
| `r.ThinOutline.SpikeFilter` | 1 | Ignores one-pixel spikes of the normal and one-pixel-thin foregrounds at sampling time. |
| `r.ThinOutline.SpatialFilter` | 1 | Pulls each pixel's slope toward the line through its neighbours. |
| `r.ThinOutline.DenseEdgeSuppression`, `.IsolatedEdgeSuppression` | 1, 1 | Don't draw an edge crowded by others on both sides; don't draw an edge shorter than three pixels. |
| `r.ThinOutline.HistoryVelocityTest` | 1 | Drops history that moved with a different object than the pixel's surface. |
| `r.ThinOutline.TileClassification` | 1 | Runs the composite and the history fetch only where there are records. |
| `r.ThinOutline.DebugView` | 0 | 1 = this frame's checks, 2 and 4 = the records, 3 = the outline alpha per axis, 6 = the listed tiles, 7 = the suppressions, 9 = the presence. |

### The Two Drawing Modes

| | Before the upscaler (0) | After the upscaler (1, default) |
|---|---|---|
| Upscaler history | the outline is accumulated, filtered and moved by TSR | not involved: each frame draws its own outline |
| Depth and velocity overwrite | yes, so silhouettes move with the foreground | none |
| Depth of field, motion blur, translucency | drawn over the outline | the outline is drawn over them, and stays sharp |
| Thickness | in display pixels, through the upscaler's filter | exactly in display pixels |
| Pixels the composite runs on | rendering pixels | display pixels: four times as many at 50% |

Drawing after the upscaler is the default because it removed the fading and
dashing that the upscaler's history caused on thin outlines; the
[drawing and performance]({% post_url 2026-10-11-thin-outline-drawing-and-performance %})
post has the comparison. Its price is that the outline is painted over fog
and translucency, since both have been composited by then. An object inside
fog gets a crisp outline drawn over the fog. Neither mode is fogged; the
before-upscaler mode sits under the "after DOF" translucency instead.

### Cost

In my test scene at 1280×720 on an RTX 5080, with every switch at its
default, the plugin's passes together take about 0.10 ms at 73% screen
percentage drawing after the upscaler, and about 0.16 ms at 100% drawing
before it (measured on 2026-10-11). The records take 44 bytes per rendering
pixel, 48 with the velocity test on, kept twice for the history: about 88 MB
at 1280×720.

## Limitations

- **Sub-pixel details.** The algorithm samples as finely as the G-buffer and
  no finer. Every check compares two samples one rendering pixel apart and
  the reconstruction assumes a full pixel-wide band around each edge in
  which that comparison fires. A thin wire, a small hole or a narrow gap
  breaks that assumption: the checks fire only on the few frames where the
  two samples happen to straddle the detail, and its edges are reconstructed
  partially or not at all. The spike filter deliberately drops geometry one
  pixel thin.
- **Statistical noise.** The lines are estimates from samples accumulated
  over time, so they carry variance. A record that has just appeared or been
  reset wobbles until enough frames have accumulated; a converged record
  still keeps a residual wobble, because the decay caps how many samples it
  effectively holds; and every yes-or-no decision made from an estimate, such
  as which axis to draw, turns a little noise near the threshold into a
  visible flip. Much of the series, from the pooling to the presence and the
  spatial filter, is about keeping that noise from turning into dashed or
  blinking outlines.
- **Fog and translucency.** Neither drawing mode is fogged, and the default
  mode draws over all translucency.
- **A small bias at 100% screen percentage.** TSR's jitter sequence has 11
  positions there, and the 11-point Halton set is not mirror-symmetric, so
  lines of one slope sign are drawn about 0.1 pixels off. Mirroring the
  jitter in a simulation moves the error to the other sign. It is a property
  of fitting lines to a short, lopsided point set, not a logic bug, and it is
  gone from 50% screen percentage on, where the sequence has 37 positions.
- **Silhouette thickness** is limited to about one rendering pixel beyond
  the edge, on the background side.
- **Dynamic resolution** resets the history whenever the view size changes.
  Split screen, scene captures and DLSS are untested.

## How I Came Up with the Algorithm

I was working on outlines, and I started with a simple G-buffer algorithm:
check the four axial neighbours of each pixel and decide whether to paint it
with the outline. To make the result look thin, I blurred it.

Then I had an idea: record the extreme position, along the inducer's axis,
at which the G-buffer check detected the edge. For the left inducer, I
recorded the minimum x at which the outline was detected, for the right
inducer the maximum x, and for the vertical inducers the same with y. The
outline was then painted only when the jittered sample position fell within
that recorded range. I asked Claude Fable to implement it.

Of course, that only worked for axis-aligned edges. A single extreme along
one axis can't describe a slanted edge, so diagonal edges came out jagged.

So I went back to the blurring method, and then found that dense outlines
flickered. My first fix was to accumulate the outline's strength over time,
as an alpha value. That made the outline worse and the jagging stronger:
every jittered sample within a rendering pixel now produced the same result,
so the upscaler had no sub-pixel detail left to reconstruct.

Next, inspired by the SVGF paper, I asked Fable to add a weighted temporal
filter: a temporal estimator tracked the variance of each pixel's outline
alpha, and the filter was applied only to pixels with high variance. (This
method is no longer used.)

Then I came up with another idea. I had been recording only the minimum or
maximum along one axis. What if I recorded both the minimum and the maximum
along the inducer's axis, together with the other coordinate at the moment
each occurred? For the left inducer, say the detected position with the
smallest x so far was (0.1, 0.2) and the one with the largest x was
(0.8, 0.7). I'd record both points. If a later sample detected the edge at
(0.9, 0.77), it would replace (0.8, 0.7). Those extreme points trace out the
parallelogram-shaped region where the edge can be detected, and from that
shape, the edge's line segment itself. To let stale records be replaced, the
recorded extremes would decay over time, the minimum creeping up and the
maximum creeping down, so an old extreme would eventually lose to a fresh
sample.

I asked Fable about this idea, and its answer came down to four points:

- A min/max record like this is statistically unstable.
- A single outlier can corrupt the record, and it may not be superseded for
  a long time.
- Why not use OLS regression instead?
- As in the SVGF paper, temporal accumulation can build unbiased estimators.

I was flabbergasted. I simply hadn't thought of OLS regression, even though
it's something you learn in a first- or second-year university statistics
course.

Fable didn't hand me a brilliant idea out of nowhere. It filled a hole. But
that hole was in a critical, load-bearing part of the idea, and it had been
making everything else creak. Looking back, it went like this:

1. When I first came up with the min/max idea, Fable didn't point out any
   problem with it.
2. In a later session, I gave Fable, or reminded it of, the knowledge of
   temporal estimators, with the SVGF paper as a reference.
3. I came up with the idea of recording the points with the minimum and
   maximum values along the inducer's axis.
4. Fable finally came up with the fix for the critical problem in my idea.

The plugin itself was then built the same way, over about two weeks: I
decided what to try and judged the results in the engine, and Fable wrote
the code, the Python mirrors of the shaders that checked it, and the notes
that these posts are drawn from.

## References

Christoph Schied, Anton Kaplanyan, Chris Wyman, Anjul Patney, Chakravarty
R. Alla Chaitanya, John Burgess, Shiqiu Liu, Carsten Dachsbacher, Aaron
Lefohn, and Marco Salvi. 2017. "Spatiotemporal Variance-Guided Filtering:
Real-Time Reconstruction for Path-Traced Global Illumination." In
*Proceedings of High Performance Graphics (HPG '17)*. ACM.
<https://doi.org/10.1145/3105762.3105770>

## How to Cite

If you reference this work, please cite this post as:

> Donguk Lim. "ThinOutline: TAA-Compatible Toon Outlines Thinner Than a
> Rendering Pixel." *Notes on Rendering* (blog), October 11, 2026.
> <https://donguklim.github.io/rendering/npr/2026/10/11/thin-outline-plugin.html>

Or in BibTeX:

{% raw %}
```bibtex
@misc{lim2026thinoutline,
  author       = {Donguk Lim},
  title        = {{ThinOutline}: {TAA}-Compatible Toon Outlines Thinner Than a Rendering Pixel},
  howpublished = {Notes on Rendering (blog)},
  year         = {2026},
  month        = oct,
  url          = {https://donguklim.github.io/rendering/npr/2026/10/11/thin-outline-plugin.html}
}
```
{% endraw %}

The [September post]({% post_url 2026-09-06-taa-compatible-toon-outline-gbuffer-contour %})
describes the first design and is kept for the record.
