---
layout: post
title: "TAA-Compatible Toon Outlines: The Pixel-Thickness Problem with G-Buffer Contours"
date: 2026-09-06
categories: [rendering, npr]
---

<!--
  Working title — feel free to rename. Other options considered:
  - "Sub-Pixel Toon Outlines Under TAA Upscaling: A G-Buffer Contour Approach"
  - "Why G-Buffer Toon Outlines Get Thicker Under TAAU / DLSS / TSR"
-->

Most real-time toon/NPR outline techniques draw contours by inspecting the
G-buffer: comparing depth, normal, and/or object-ID samples between
neighboring pixels and flagging an edge wherever the discontinuity crosses
some threshold. That works well in isolation, but it runs into a subtle
problem once a temporal upscaler — TAAU, DLSS, FSR2, XeSS, Unreal's TSR, and
similar — is added to the pipeline.

These upscalers render the scene at a reduced internal resolution (the
**screen percentage**, or render scale) and reconstruct a full
display-resolution image from it using temporal accumulation. A G-buffer
contour algorithm, at its core, is just an edge *detector*: for a given
render-resolution pixel, it answers "does a contour pass through here?" Since
the G-buffer only has samples at render resolution, the thinnest outline such
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

## What this post covers

_(placeholder — fill in)_

- Why this matters for a toon/cel-shading art style specifically, beyond just "aliasing"
- The G-buffer contour detection method used here (depth / normal / ID edge tests)
- How the outline is generated and composited pre- vs. post-TAA, and why that ordering matters
- The fix/approach for keeping outline thickness consistent (or intentionally scaled) across screen percentages
- Results: outline thickness at 100%, 75%, 50% screen percentage, with and without the fix

## Background

_(placeholder — fill in)_

## Algorithm

_(placeholder — fill in)_

## Results

_(placeholder — fill in)_
