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


## Core Idea

### The Goal: Reconstruct the Edge, Not Just Detect It

Everything above treats the G-buffer algorithm as a binary detector: does a
contour cross this one-render-pixel-long segment, yes or no. The goal of
this algorithm is to go a step further and reconstruct the actual
line-segment equation of the edge or contour near each render pixel,
instead of just detecting whether one exists. Concretely, the edge is
reconstructed as $y = a + bx$ in that pixel's local space — coordinates
measured relative to the render pixel itself, rather than screen space —
which is also the form the OLS fit further down solves for.

Once you have that line equation, rendering the outline becomes a distance
check: draw the outline at a render pixel only when the distance from that
pixel's jittered sample position to the reconstructed edge line falls within
some desired thickness. Because that threshold is a property of the
reconstructed edge rather than of the render-pixel grid, it can be set
arbitrarily thin — even below the display pixel's width — and it stays
consistent regardless of the upscaler's screen percentage. That's the
sub-render-pixel-accurate, TAA-compatible outline this post opened with.

### What G-Buffer Algorithms Do

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

Let's ignore the top and bottom comparisons of the G-buffer algorithm and
just look at the left and right neighbor comparisons. The edge line segment
splits the parallelogram into two one-pixel-wide halves: the left half
detects the edge when a pixel's jittered sample is compared with its right
neighbor, and the right half detects it when compared with its left
neighbor.

![Same three-pixel row with only the left half of the parallelogram drawn — a one-render-pixel-wide band bounded by the edge on its right. A green pair (left pixel sample, center pixel sample) and a yellow pair (center pixel sample, right pixel sample), each at the same local jitter offset, both land inside the band and are connected by an arrow from the left sample to the right sample, showing both are detected against their right neighbor](/assets/images/taa-toon-outline/edge-detectable-neighbor-pairs.svg)

The same thing holds on the other side, mirrored: the right half of the
parallelogram is the region where a pixel's jittered sample detects the edge
against its *left* neighbor.

![Same three-pixel row, but now the hatched band is the right half of the parallelogram, bounded by the edge on its left. A green pair (center pixel sample, left pixel sample) and a yellow pair (right pixel sample, center pixel sample), each at a different local jitter offset than the previous figure, both land inside this band and are connected by an arrow from the right sample to the left sample, showing both are detected against their left neighbor](/assets/images/taa-toon-outline/edge-detectable-neighbor-pairs-left.svg)

And, just as important, here's what it looks like when the edge *isn't*
detected: when a pair's shared jitter offset falls outside the band, both
samples land on the same surface, so the comparison finds no discontinuity
at all.

![Same three-pixel row and right-neighbor band as before, but now a teal pair (left pixel sample, center pixel sample) and an amber pair (center pixel sample, right pixel sample) both fall outside the hatched band — each pair's samples land on the same surface (both purple or both gray), so neither pair detects a contour against the right neighbor](/assets/images/taa-toon-outline/edge-not-detected-right.svg)

![Same three-pixel row and left-neighbor band as before, but now a teal pair (center pixel sample, left pixel sample) and an amber pair (right pixel sample, center pixel sample) both fall outside the hatched band — each pair's samples land on the same surface, so neither pair detects a contour against the left neighbor](/assets/images/taa-toon-outline/edge-not-detected-left.svg)


So, for each neighbor-comparison direction, the detected/not-detected jitter
samples accumulated across many frames give you the data to infer the shape
of that direction's parallelogram.

### Using Statistical Methods with a Temporal Record

You can fit that data with a statistical method like [ordinary least squares
regression](https://en.wikipedia.org/wiki/Ordinary_least_squares) to find
the line running through the middle of the detected samples, or
[linear discriminant analysis](https://en.wikipedia.org/wiki/Linear_discriminant_analysis)
to find the boundary between the detected and not-detected samples instead.
Either way, you end up with an estimate of the edge's position and
orientation that's noticeably more accurate than any single frame's
one-pixel-wide detection.

![Same three-pixel row and left-neighbor band as before, but the colored sample pairs are replaced with many small black dots scattered across the hatched band, accumulated over many frames. A red trend line fitted through these dots via OLS regression runs through the middle of the parallelogram, parallel to the true black edge line](/assets/images/taa-toon-outline/ols-edge-fit-left.svg)

LDA takes a different approach: instead of fitting through the middle of one
class, it looks for the boundary that best separates two labeled classes of
points. Label each accumulated jitter sample by whether that frame's
comparison detected a contour or not, and LDA gives you back the line that
best separates the "detected" cluster from the "not detected" one. Since the
detectable band has two sides, running this twice — once against the
samples just outside its near edge, once against the samples just outside
its far edge — recovers both boundaries of the band. One of those boundaries
is, by construction, the true geometric edge itself.

![Same three-pixel row and right-neighbor band as before. Black dots fill the parallelogram (detected samples) while red dots sit outside it on both sides (not-detected samples, one cluster on the near/purple side and one on the far/gray side). Two green dashed lines, fitted via LDA, trace the band's two boundaries — the near one has no corresponding real edge, while the far one lands exactly on the true black edge line](/assets/images/taa-toon-outline/lda-edge-fit-right.svg)

This sounds nice, but it seems like you'd need to hold onto every sampled
point to compute either statistic. Actually, you don't — both OLS and LDA
can be computed from a handful of running sums, so you can apply them to
data that's accumulated temporally, with decay, instead of keeping every
sample around.

#### Ex. OLS with Temporal Accumulation

As a quick illustration of that pattern, here's what it looks like for OLS
specifically — I'll go into the full implementation, as a dedicated section
of its own, later in this post.

Given accumulated samples $(x_i, y_i)$, OLS fits the line $y = a + bx$ using
only four running statistics — $E[x]$, $E[y]$, $E[x^2]$, and $E[xy]$ — each
of which can be maintained as a decayed running average, updated by one new
sample per frame:

$$
b = \frac{E[xy] - E[x]\,E[y]}{E[x^2] - E[x]^2}, \qquad a = E[y] - b\,E[x]
$$


And each of those four expectations can be maintained without needing to
store any history, by keeping a single decayed running value per variable
and updating it once per frame:

$$
\langle v \rangle_i = (1-d)\,\langle v \rangle_{i-1} + d\, v_i,
\qquad v \in \{\,x,\ y,\ x^2,\ xy\,\}
$$

where $v_i$ is that frame's new sample for whichever variable — $x$, $y$,
$x^2$, or $xy$ — and $d \in (0,1)$ is the decay rate. After enough frames,
$\langle v \rangle_i$ tracks $E[v]$ closely enough to plug directly into the
OLS formula above.


## Actual Implementation with Temporal OLS

My first attempt implemented the core idea using OLS.

Each render pixel maintains a historically accumulated record of the
required data for each of its four edge-inducing directions (top, bottom,
left, right). Each of the four running statistics — $E[x]$, $E[y]$,
$E[x^2]$, $E[xy]$ — is stored as a single `float4`, one component per
inducer direction.

The first problem I ran into was that the sampled data gets cut off at the
pixel boundary.

For a given inducer direction, an edge produces edge-detected jitter
positions across two pixels: the pixel that actually contains the edge, and
the pixel on the opposite side from the inducer direction.

Once you know which of the two is the containing pixel, you can simply
merge the other pixel's data into it. This is doable as long as the edge is
isolated — no other edge within a radius of two render pixels. For each
inducer direction, you just check whether the neighboring pixel in that
direction also has data for the same inducer direction.

However, what if two edges are less than two render pixels apart? This
would leave more than two consecutive pixels holding left-induced edge
data.

![Four horizontally adjacent pixels. A second edge, with a slightly different slope, sits inside the first pixel, and the original edge sits inside the third pixel. Each edge has its own one-render-pixel-wide, left-neighbor-detectable band — the first edge's band overlaps the first and second pixels, the second edge's band overlaps the third and fourth. All four pixels are labeled "left edge detected"](/assets/images/taa-toon-outline/two-edges-left-detection.svg)


Well, the solution turns out to be simple: don't merge neighboring pixels'
data at all. Instead, apply OLS to each render pixel independently, using
only the data built from that pixel's own jitter samples.

This yields a line that runs not through the middle of the full
parallelogram, but through the middle of the trapezoid formed by cutting
that parallelogram at the pixel boundary.

![Same four-pixel setup as before, but now four red line segments are added, one per pixel, each running through the middle of that pixel's own trapezoid — the piece of its edge's detectable parallelogram left after the vertical pixel-boundary cut. The four red lines are visibly different from each other and from the two black true-edge lines, since each pixel only fit its own half of the data](/assets/images/taa-toon-outline/two-edges-per-pixel-trapezoid-fit.svg)

Even so, assuming the accumulated data is accurate enough, you can
reconstruct the true edge segment from these per-pixel fits.

A pixel that doesn't contain the actual edge will always have an intercept
less than 0.5, so checking the intercept alone tells you whether a given
pixel contains the edge or not.

![Same four-pixel setup as before, but the per-pixel labels now show the intercept test instead of the detection outcome: the first and third pixels — which actually contain an edge — are labeled "intercept ≥ 0.5", while the second and fourth pixels — which only detect their neighbor's edge without containing it — are labeled "intercept ≤ 0.5"](/assets/images/taa-toon-outline/two-edges-per-pixel-intercept.svg)

For the edge-containing pixel, you're left with the line equation that runs
through the middle of its trapezoid — and from there, simple algebra
reconstructs the actual edge equation.

Work in local pixel coordinates, $x, y \in [0, 1]$, and parametrize the edge
as $x = A + By$ rather than $y = A + Bx$ — every edge in this post enters
through the top of its pixel and exits through the bottom, so solving for
$x$ as a function of $y$ avoids the near-vertical slope that form would
otherwise have. For the edge-containing pixel, the left trapezoid is the
region $0 \le x \le A + By$, so at each height $y$ its horizontal slice runs
from $0$ to $A + By$, and the midpoint of that slice is $(A+By)/2$. The line
through the middle of the trapezoid is exactly the line through all of
those midpoints:

$$
x = \frac{A + By}{2} = \underbrace{\frac{A}{2}}_{a} + \underbrace{\frac{B}{2}}_{b}\,y
$$

So if $x = a + by$ is the line through the middle of the trapezoid, the
actual edge is just that line doubled:

$$
A = 2a, \qquad B = 2b \qquad \Longrightarrow \qquad x_{\text{edge}} = 2a + 2by
$$


### Inducer Direction Selection

So each pixel ends up with four reconstructed edges, one per inducer
direction. My shader picks whichever one has the lower combined score of
two terms: the OLS [SDE](https://en.wikipedia.org/wiki/Reduced_chi-squared_statistic),
and the area between the reconstructed line segment and the pixel boundary
on the inducer side.

The SDE, scaled by a custom factor, is added to that area to form the final
criterion value.

I used the area term because I wanted to favor whichever edge stays more
closely aligned with — sticks more closely to — its inducer-direction axis.

### Spatial Filter

My first attempt got outlines as thin as one display pixel. However, the
outline color faded intermittently along the contour as the screen
percentage dropped.

I eventually realized why: I'd only reconstructed edges that are temporally
stable within each pixel, but not spatially stable across neighboring
pixels.

So I added a spatial filter that blends outlines across neighboring pixels.

For a given target pixel, extend its reconstructed edge outward in both
directions and find whichever neighboring pixel it overlaps the most in
each direction. Then blend the historical estimator variables between the
two pixels to produce a blended line.

To do that, I also keep a separate decayed running count of edge-detected
jitter positions.

This fixed the outline color fading.

#### Visualization Example

![A 3×3 grid of pixels. The center (target) pixel contains its own reconstructed edge, a slanted line segment](/assets/images/taa-toon-outline/spatial-filter-target-edge.svg)

![Same 3×3 grid. The target pixel's edge is now extended outward (dashed) in both directions until it reaches the grid's outer boundary, landing inside the top-left and bottom-right diagonal neighbor pixels](/assets/images/taa-toon-outline/spatial-filter-extend-edge.svg)

Select the two most overlapping pixels in each extended direction.

![Same 3×3 grid, with the top-left and bottom-right pixels now colored to show they've been selected as the most overlapping neighbor in each extended direction](/assets/images/taa-toon-outline/spatial-filter-selected-neighbors.svg)

Fetch the line segments in the selecting neighboring pixels.

![Same 3×3 grid, but now the top-left and bottom-right neighbor pixels also show their own reconstructed edges — each with a slightly different slope and intercept from the target pixel's edge](/assets/images/taa-toon-outline/spatial-filter-neighbor-edges.svg)

Blend line segments of the selected neighbors.

![Same 3×3 grid, with one continuous blue line now drawn across it, built by blending the three black line segments' historical estimator variables — the blue line threads through all three rather than matching any single one exactly](/assets/images/taa-toon-outline/spatial-filter-blended-line.svg)


## Limitation of the Algorithm

This outline-reconstruction algorithm's sampling interval is limited by the
G-buffer's resolution. If there's a hole smaller than one pixel, the edge it
creates is only detected when one of the two compared jitter positions
lands inside the hole and the other doesn't.

## Further Improvements

I have a few improvement ideas in mind — this algorithm is still under
active development.

### Data Accumulation

#### Don't Discard Not-Detected Jitter Positions

My current implementation just discards jitter positions where no edge was
detected — that's wasted statistical data.

I'm thinking of running two separate OLS fits per inducer direction: one
over the edge-detected samples, one over the not-detected samples — and
then blending the two resulting line segments.

#### Don't Cut Statistical Data at the Pixel Boundary

I said earlier that you can just apply OLS independently for each pixel,
but that's also an act of discarding statistical data. I should combine the
statistical data whenever it's obvious no overlap can occur — for instance,
when only two consecutive pixels share the same inducer-direction data.

#### What Would This Improvement Give Us?

It would make the reconstructed line segment converge faster under
temporal accumulation.

### Spatial Filter

The spatial filter I used here is something I put together quickly —
there's probably a better approach, and I should do more research into it.

Currently, the spatial filter has improved the outline color fading at
lower screen percentages, but hasn't eliminated it completely.


