---
layout: post
title: "TAA-Compatible Toon Outlines: The Pixel-Thickness Problem with G-Buffer Contours"
date: 2026-09-06
categories: [rendering, npr]
---


## Background

One common approach to toon/NPR outlines is to locate contours by inspecting
G-buffer data — comparing depth, normal, and/or object-ID samples between
neighboring pixels and flagging an edge wherever the discontinuity crosses
some threshold. This is different from geometry-based approaches like
inverted hull, which draw the outline as an actual mesh and don't have the
problem described below — they're inherently TAA-compatible. G-buffer-based
detection works well without a temporal upscaler, but runs into a subtle
problem once one — TAAU, DLSS, FSR2, XeSS, Unreal's TSR, and similar — is
added to the pipeline.

These upscalers render the scene at a reduced internal resolution (the
**screen percentage**, or render scale) and reconstruct a full
display-resolution image from it using temporal accumulation. A G-buffer
contour algorithm, at its core, is just an edge *detector*: for a given
render-resolution pixel, it answers "does a contour pass through this pixel?"
or "does a contour pass between these two pixels?"

Since the G-buffer only has samples at render resolution, the thinnest outline such
an algorithm can ever produce is exactly **one render pixel wide** — there is
no sub-pixel information to draw anything thinner from.

That constraint becomes visible the moment the image is upscaled. If the
renderer runs at 50% screen percentage, the upscaler doubles the linear
dimensions when reconstructing the display-resolution frame. A contour that
was already at the minimum representable width of one render pixel gets
stretched along with everything else, and comes out at roughly
`1 / screen_percentage` display pixels wide — two display pixels at 50%
screen percentage, four at 25%, and so on:

![A one-render-pixel-wide G-buffer contour becomes two display pixels wide after upscaling from 50% screen percentage](/assets/images/taa-toon-outline/pixel-thickness-upscale.svg)

In other words, the thinnest outline the algorithm can draw gets *thicker*,
in display pixels, as render resolution drops relative to display
resolution — the opposite of what you'd want from an outline style that's
supposed to stay crisp and resolution-independent.


A simple, standard compromise is to blur the render-resolution outline by an
amount that scales with the screen percentage, or to supersample the depth
and normal G-buffer data. Now, in this post, I'm proposing an algorithm that
keeps the outline crisp without those trade-offs — see below for the core
idea.

### How Do Temporal Upscalers Reconstruct Sub-Render-Pixel Detail

Temporal upscalers don't get extra detail from any single frame — a single
frame is still only rendered, and G-buffer-sampled, at render resolution.
What they add is time.

Each frame, the camera's projection is offset by a small **sub-pixel
jitter** — typically a low-discrepancy sequence like a Halton sequence,
cycling through a handful of offsets that together cover the pixel footprint
fairly evenly. So frame *N* samples the scene at one sub-pixel position
within each render pixel, frame *N+1* samples it at a different sub-pixel
position, and so on. The upscaler then reprojects previous frames into the
current frame using per-pixel motion vectors and blends them with the
current frame's jittered sample into a **history buffer**. After a handful
of frames, that history buffer holds several samples per render pixel — this
jitter-and-accumulate loop is basically how temporal upscalers scan out
sub-render-pixel detail over time, rather than any single frame having more
resolution than the G-buffer it was built from.

![Four frames each sample the scene at a different jittered sub-pixel offset; accumulating them builds up several sub-pixel samples inside one render pixel](/assets/images/taa-toon-outline/jitter-accumulation.svg)

This is why geometry-based outline techniques like inverted hull don't have
the thickness problem described above: an outline mesh is rasterized like
any other triangle in the scene, at whatever jittered sub-pixel offset that
frame happens to use. Its silhouette edge lands at a slightly different
sub-pixel position each frame, gets accumulated the same way ordinary
geometry edges do, and comes out anti-aliased and sub-render-pixel-accurate
after temporal accumulation — with no extra work.

A G-buffer contour detector doesn't get that benefit for free. Even though
the underlying G-buffer samples are taken at a jittered sub-pixel offset
each frame, the detector's output — is there a contour here, yes or no — is
still a value computed once per render pixel, on a fixed pixel grid. There's
no continuous edge for the accumulation to refine the position of; the
history buffer just receives four frames' worth of "yes, this pixel is on a
contour," instead of four different sub-pixel edge positions the way a
rasterized triangle edge would produce. That mismatch — a discrete,
grid-locked detector feeding into a system built to accumulate continuous
sub-pixel geometry — is the root of the pixel-thickness problem.

## What G-Buffer Algorithms Do

Take a really simple G-buffer algorithm as an example. For a given target
pixel, it checks whether the depth or normal value differs from each of its
four axial neighbors — up, down, left, right — by more than some threshold.
If the difference exceeds the threshold against any one of those neighbors,
the target pixel is marked as containing an outline.

Marking both sides of every discontinuity like this produces an outline
that's two render pixels thick. You can narrow that down to one render
pixel by only drawing the outline on whichever side of the pair has the
higher depth (i.e. is farther from the camera), or the more camera-facing
normal.

Here's the important part: what the algorithm is actually doing is checking
whether an edge exists between the *jittered sample positions* of the
target pixel and each neighbor — not between the pixels' centers or their
full footprints. So, effectively, it's detecting whether an edge crosses the
one-render-pixel-long line segment connecting the target's jittered sample
to that neighbor's jittered sample, because G-buffer data is collected from the jittering positions.

![Five pixels in a plus shape, each with a dot at the same local jitter offset. The geometric edge sits between the left and target pixel and crosses the check segment between their sample dots, so that segment is highlighted red as a detected contour, while the other three check segments stay gray for no contour](/assets/images/taa-toon-outline/gbuffer-edge-check.svg)


So, given an actual geometric edge — the line along which the real
discontinuity occurs — the set of jitter offsets that would place it between
two neighboring pixels forms a parallelogram: two render pixels wide,
running parallel to the edge, with the edge itself as its middle line. If
the jitter offset for that frame falls inside this parallelogram, the edge
is detected between the two pixels; otherwise, it isn't.

![Three horizontally adjacent pixels with a slanted geometric edge through the center pixel, and a hatched parallelogram straddling it as the edge-detectable area — the edge runs through the middle of the parallelogram, and a jittered sample pair only detects the edge if its shared offset falls inside the hatched band](/assets/images/taa-toon-outline/edge-detectable-parallelogram.svg)

## Core Idea of Algorithm

_(placeholder — fill in)_

## Results

_(placeholder — fill in)_
