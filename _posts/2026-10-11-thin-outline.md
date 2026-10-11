---
layout: post
title: "TAA-Compatible Thin Toon Outlines From G-Buffer Contours"
date: 2026-10-11 16:00:00 +0900
categories: [rendering, npr]
---

This is an algorithm for drawing toon outlines from G-buffer data alone that
are thinner than one rendering pixel and stay crisp under a temporal upscaler
such as TSR. This post states the problem, gives the idea in short, and links
to the posts that explain each part; the parts are separate pages only
because one page would be uncomfortable to read. It replaces my
[post from September]({% post_url 2026-09-06-taa-compatible-toon-outline-gbuffer-contour %}),
which described the first design; the algorithm changed in several places
while I implemented it, and this series describes what it does now.

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

## The Idea in Short

The algorithm turns the detector's yes-or-no answers into a line. Each frame,
the G-buffer is sampled at that frame's jittered sub-pixel position, so each
frame's detection says whether an edge passes between two jittered sample
points. Over the frames of the upscaler's jitter sequence, those answers
trace out the shape of the region where the edge is detectable, and from
that region each rendering pixel reconstructs the edge's line equation by
ordinary least squares, from a handful of running statistics it keeps from
frame to frame. The line is then drawn with any thickness you like, measured
in display pixels, with exact coverage per pixel. The outline no longer
depends on the screen percentage.

Two kinds of edges are drawn: **creases**, where the normal changes abruptly
on a continuous surface, and **silhouettes**, where the depth jumps between a
foreground and a background. Both are reconstructed the same way, with one
record per pixel and per inducer axis (horizontal: the left and right checks;
vertical: the top and bottom checks), and each record is either a crease or a
silhouette record.

Per frame, the work falls into three steps:

1. **Recording**, at rendering resolution, before the upscaler. Every
   rendering pixel fetches last frame's records at the position the motion
   vectors point to, tests whether they still belong to the same surface,
   decays them, runs this frame's crease and silhouette checks on the
   jittered G-buffer, and adds the samples that fired to the records.
2. **Reconstruction**, once per rendering pixel. Each pixel pools its record
   with its neighbours', fits the edge, picks between the horizontal and
   vertical candidate, filters the line against its neighbours', suppresses
   edges that are crowded or too short, and scales the edge's strength by how
   consistently it has been found. The result is one line per rendering
   pixel.
3. **Drawing.** Each display pixel is painted by the exact area of the
   outline band around that line inside it. This can happen after the
   upscaler, at display resolution, or before it, at rendering resolution,
   where the upscaler then has to be told the foreground's depth and
   velocity under silhouette pixels so that it moves the outline with the
   object in front.

## The Parts

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
  why blending neighbouring lines fails on a static edge, and the slope
  through the centre crossings of the neighbours that is fitted instead.
- [Flicker suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %}):
  what still flickers after the upscaler and why, measured over jitter
  cycles, the dense and isolated edge suppression that were tried against
  it first, and the presence and spike filter that remove most of it.
- [Tile classification]({% post_url 2026-10-11-thin-outline-tile-classification %}):
  running the drawing passes only on the tiles within reach of a record,
  and why the passes that remained were waiting on their loads rather than
  doing too much.
- [Drawing and performance]({% post_url 2026-10-11-thin-outline-drawing-and-performance %}):
  coverage of the display pixel, drawing before the upscaler against after
  it, and what the implementation costs and why.

## The Implementation

I implemented the algorithm as an Unreal Engine plugin, ThinOutline, against
a source build of 5.8:
<https://github.com/donguklim/thin-outline-plugin>. The posts quote short
pieces of its HLSL where a formula is clearer as code, and the numbers they
give were measured with it, under TSR, in the engine's third-person
template, the City Sample and Stack O Bot. Its settings, drawing modes and
costs are described in the
[drawing and performance]({% post_url 2026-10-11-thin-outline-drawing-and-performance %})
post. The algorithm needs a temporal upscaler whose jitter is uniform over
the pixel, as TSR's Halton sequence is; without temporal anti-aliasing every
sample sits at the pixel centre and nothing can be reconstructed, and a
Gaussian jitter such as legacy TAA's biases the reconstruction.

## Limitations

- **Sub-pixel details.** The algorithm samples as finely as the G-buffer and
  no finer. Every check compares two samples one rendering pixel apart and
  the reconstruction assumes a full pixel-wide band around each edge in
  which that comparison fires. A thin wire, a small hole or a narrow gap
  breaks that assumption: the checks fire only on the few frames where the
  two samples happen to straddle the detail, and its edges are reconstructed
  partially or not at all. Geometry one pixel thin is deliberately dropped at
  sampling time.
- **Statistical noise.** The lines are estimates from samples accumulated
  over time, so they carry variance. A record that has just appeared or been
  reset wobbles until enough frames have accumulated; a converged record
  still keeps a residual wobble, because the decay caps how many samples it
  effectively holds; and every yes-or-no decision made from an estimate, such
  as which axis to draw, turns a little noise near the threshold into a
  visible flip. Much of the series, from the pooling to the presence and the
  spatial filter, is about keeping that noise from turning into dashed or
  blinking outlines.
- **Fog and translucency.** Drawing after the upscaler paints the outline
  over everything composited before it, fog and translucency included;
  drawing before it sits under some of the translucency but is not fogged
  either.
- **A small bias at 100% screen percentage.** TSR's jitter sequence has 11
  positions there, and the 11-point Halton set is not mirror-symmetric, so
  lines of one slope sign are drawn about 0.1 pixels off. Mirroring the
  jitter in a simulation moves the error to the other sign. It is a property
  of fitting lines to a short, lopsided point set, not a logic bug, and it is
  gone from 50% screen percentage on, where the sequence has 37 positions.
- **Silhouette thickness** is limited to about one rendering pixel beyond
  the edge, on the background side.

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

The implementation was then built the same way, over about two weeks: I
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

> Donguk Lim. "TAA-Compatible Thin Toon Outlines From G-Buffer Contours."
> *Notes on Rendering* (blog), October 11, 2026.
> <https://donguklim.github.io/rendering/npr/2026/10/11/thin-outline.html>

Or in BibTeX:

{% raw %}
```bibtex
@misc{lim2026thinoutlines,
  author       = {Donguk Lim},
  title        = {{TAA}-Compatible Thin Toon Outlines From {G}-Buffer Contours},
  howpublished = {Notes on Rendering (blog)},
  year         = {2026},
  month        = oct,
  url          = {https://donguklim.github.io/rendering/npr/2026/10/11/thin-outline.html}
}
```
{% endraw %}

The [September post]({% post_url 2026-09-06-taa-compatible-toon-outline-gbuffer-contour %})
describes the first design and is kept for the record.
