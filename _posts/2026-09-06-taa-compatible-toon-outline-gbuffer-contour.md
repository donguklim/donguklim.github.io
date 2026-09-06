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

### How Does Temporal Upscalers Reconstruct Sub-Rendering Pixel details



## What G-Buffer Algorithms Do

## Core Idea of Algorithm

_(placeholder — fill in)_

## Results

_(placeholder — fill in)_
