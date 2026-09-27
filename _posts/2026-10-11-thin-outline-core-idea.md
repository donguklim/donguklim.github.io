---
layout: post
title: "ThinOutline: The Core Idea"
date: 2026-10-11 10:00:00 +0900
categories: [rendering, npr]
---

The ThinOutline plugin draws toon outlines from the G-buffer that are thinner
than a rendering pixel and stay crisp under a temporal upscaler. This post is
the core of how it does that: a G-buffer edge check can only say whether an
edge passes between two rendering pixels, but accumulated over the jittered
frames of a temporal upscaler, those yes-or-no answers pin the edge down to a
line equation accurate to a fraction of a pixel. It is the first part of a
[series on the plugin]({% post_url 2026-10-11-thin-outline-plugin %}); the
main post has the overview and the links to the other parts.

Two words I use throughout, as the plugin's code does: a **rendering pixel**
is a pixel of the renderer before the temporal upscaler, and a **display
pixel** is a pixel after it. At 50% screen percentage one rendering pixel
covers 2×2 display pixels.

## The Pixel-Thickness Problem

One common approach to toon outlines is to locate contours by inspecting
G-buffer data: compare depth, normal or object-ID samples between
neighbouring pixels and flag an edge wherever the discontinuity crosses a
threshold. Without a temporal upscaler that works well. With one (TAAU,
TSR, DLSS, FSR, XeSS) it runs into a problem.

These upscalers render the scene at a reduced internal resolution, the
**screen percentage**, and reconstruct a full display-resolution image by
temporal accumulation. A G-buffer contour algorithm is, at its core, an edge
*detector*: for a given rendering pixel it answers "does a contour pass
between this pixel and its neighbour?" Since the G-buffer only has samples at
rendering resolution, the thinnest outline such an algorithm can produce is
one rendering pixel wide. After upscaling, that is about
`1 / screen_percentage` display pixels: two display pixels at 50% screen
percentage, four at 25%.

![A one-render-pixel-wide G-buffer contour becomes two display pixels wide after upscaling from 50% screen percentage](/assets/images/taa-toon-outline/pixel-thickness-upscale.svg)

The usual compromises are to blur the rendering-resolution outline by an
amount that scales with the screen percentage, or to supersample the depth
and normal data. The plugin does neither.

### How Temporal Upscalers Reconstruct Sub-Pixel Detail

A temporal upscaler doesn't get extra detail from any single frame. What it
adds is time. Each frame, the camera's projection is offset by a small
**sub-pixel jitter**, typically a low-discrepancy sequence such as Halton,
cycling through a set of offsets that together cover the pixel footprint
evenly. Frame *N* samples the scene at one sub-pixel position inside each
rendering pixel, frame *N+1* at another, and so on. The upscaler reprojects
the previous frames into the current one with per-pixel motion vectors and
blends them with the current frame's jittered sample into a history buffer.
After a handful of frames, the history holds several samples per rendering
pixel.

![Four frames each sample the scene at a different jittered sub-pixel offset; accumulating them builds up several sub-pixel samples inside one render pixel](/assets/images/taa-toon-outline/jitter-accumulation.svg)

This is why geometry-based outlines such as an inverted hull don't have the
thickness problem: the outline mesh is rasterized like any other triangle, at
whatever jittered offset the frame uses, so its silhouette lands at a
slightly different sub-pixel position each frame and the accumulation refines
it. A G-buffer detector's output, "yes, this pixel is on a contour", is a
value computed once per rendering pixel on a fixed grid. There is no
continuous edge for the accumulation to refine; the history just receives
several frames' worth of the same yes. That mismatch, a grid-locked detector
feeding a system built to accumulate continuous geometry, is the root of the
problem.

## Reconstruct the Edge, Not Just Detect It

The goal of the algorithm is to go one step further than detection and
reconstruct the actual line equation of the edge near each rendering pixel.
Concretely, the edge is reconstructed in the pixel's local coordinates, as a
line crossing the pixel's square. Once you have that line, drawing the
outline becomes geometry: paint the band of the chosen thickness around the
line, with exact coverage per pixel. The thickness is a property of the
reconstructed line, not of the rendering-pixel grid, so it can be set
arbitrarily thin, below the display pixel if you like, and it stays the same
regardless of the screen percentage.

The rest of this post is about where the line comes from.

## What a G-Buffer Check Really Tests

Take the simplest G-buffer algorithm as the example. For a target pixel, it
checks whether the depth or normal differs from each of its four axial
neighbours, up, down, left and right, by more than a threshold. If any
neighbour differs, the target is marked as containing an outline.

Here's the important part: because the G-buffer is sampled at the frame's
jittered position, what the check actually tests is whether an edge lies
between the *jittered sample positions* of the target and its neighbour, not
between the pixels' centres or their full footprints. Every pixel in a frame
shares the same jitter offset, so the two samples are exactly one rendering
pixel apart, and the check detects whether an edge crosses the
one-pixel-long segment connecting them.

![Five pixels in a plus shape, each with a dot at the same local jitter offset. The geometric edge sits between the left and target pixel and crosses the check segment between their sample dots, so that segment is highlighted red as a detected contour, while the other three check segments stay gray for no contour](/assets/images/taa-toon-outline/gbuffer-edge-check.svg)

So, given an actual geometric edge, the set of jitter offsets that would make
it fall between two horizontally neighbouring pixels forms a
**parallelogram**: two rendering pixels wide, running parallel to the edge,
with the edge as its middle line. If the frame's jitter offset falls inside
the parallelogram, the edge is detected between the two pixels; otherwise it
isn't.

![Three horizontally adjacent pixels with a slanted geometric edge through the center pixel, and a hatched parallelogram straddling it as the edge-detectable area — the edge runs through the middle of the parallelogram, and a jittered sample pair only detects the edge if its shared offset falls inside the hatched band](/assets/images/taa-toon-outline/edge-detectable-parallelogram.svg)

Look at just the left and right comparisons. The edge splits the
parallelogram into two one-pixel-wide halves. The left half is where a
pixel's sample detects the edge when compared with its *right* neighbour,
the right half where it detects the edge against its *left* neighbour. I'll
call these the **right check** and the **left check** of the pixel, after
the neighbour they compare with, and the pair of them the pixel's
**horizontal axis**; the top and bottom checks make up its **vertical
axis**.

![Same three-pixel row with only the left half of the parallelogram drawn — a one-render-pixel-wide band bounded by the edge on its right. A green pair (left pixel sample, center pixel sample) and a yellow pair (center pixel sample, right pixel sample), each at the same local jitter offset, both land inside the band and are connected by an arrow from the left sample to the right sample, showing both are detected against their right neighbor](/assets/images/taa-toon-outline/edge-detectable-neighbor-pairs.svg)

![Same three-pixel row, but now the hatched band is the right half of the parallelogram, bounded by the edge on its left. A green pair (center pixel sample, left pixel sample) and a yellow pair (right pixel sample, center pixel sample), each at a different local jitter offset than the previous figure, both land inside this band and are connected by an arrow from the right sample to the left sample, showing both are detected against their left neighbor](/assets/images/taa-toon-outline/edge-detectable-neighbor-pairs-left.svg)

And when the jitter falls outside the band, both samples of a pair land on
the same surface and the check finds nothing:

![Same three-pixel row and right-neighbor band as before, but now a teal pair (left pixel sample, center pixel sample) and an amber pair (center pixel sample, right pixel sample) both fall outside the hatched band — each pair's samples land on the same surface (both purple or both gray), so neither pair detects a contour against the right neighbor](/assets/images/taa-toon-outline/edge-not-detected-right.svg)

So for each check, the detected and not-detected jitter offsets accumulated
over many frames carry the shape of that check's half of the parallelogram,
and the parallelogram's middle line is the edge.

### The Checks the Plugin Runs

The plugin's checks are a little more careful than "does the normal differ",
but they are still checks between a target pixel C and an axial neighbour N,
helped by the opposite neighbour O. A **crease** check compares world-space
normals. Its measure is

$$
\min\big(\,\lvert\, \mathbf n_C \times \mathbf n_N - \mathbf n_O \times \mathbf n_C \,\rvert,\ \lvert\, \mathbf n_C \times \mathbf n_N \,\rvert\,\big),
$$

the change of the normal toward N, minus the change from O to C where that
is comparable: on a smoothly curving surface the two changes are similar and
cancel, so only an abrupt bend counts. The check is skipped across a depth
discontinuity, and it fires when the measure exceeds a ridge or valley
threshold (0.25 each by default), with a strength that rises linearly above
the threshold and saturates at 1. A **silhouette** check compares depths,
with a measure built from world positions that stays zero on a plane seen at
any angle; the [silhouettes]({% post_url 2026-10-11-thin-outline-silhouettes %})
post describes it. A pixel with a silhouette toward any neighbour runs no
crease checks at all. Everything below applies to both kinds; where
silhouettes differ, that post says so.

The sample position itself is the same for every pixel of a frame. The
engine's projection jitter moves the scene by the jitter vector, so every
pixel samples the G-buffer at the local position 0.5 minus this frame's
jitter.

## Fitting the Band With a Temporal Record

You can fit the accumulated jitter offsets with a statistical method such as
[ordinary least squares regression](https://en.wikipedia.org/wiki/Ordinary_least_squares)
(OLS) to find the line running through the middle of the detected offsets,
or [linear discriminant analysis](https://en.wikipedia.org/wiki/Linear_discriminant_analysis)
to find the boundary between the detected and not-detected ones. Either way,
the estimate of the edge's position and orientation is far more accurate
than any single frame's one-pixel-wide detection. The plugin uses OLS; the
rest of this post is about making it work.

![Same three-pixel row and left-neighbor band as before, but the colored sample pairs are replaced with many small black dots scattered across the hatched band, accumulated over many frames. A red trend line fitted through these dots via OLS regression runs through the middle of the parallelogram, parallel to the true black edge line](/assets/images/taa-toon-outline/ols-edge-fit-left.svg)

This sounds like it needs every sampled point. It doesn't. OLS is a function
of a handful of running statistics, so it can be computed from data
accumulated temporally, with decay, without keeping any sample around. Work
in the pixel's local coordinates with $u \in [0, 1]$ along the axis and
$v \in [0, 1]$ across it, and parametrize a line as $u = a + bv$: the edges
an axis cares about run roughly across it, so solving for $u$ as a function
of $v$ avoids near-vertical slopes. Given samples $(u_i, v_i)$, the OLS fit
is

$$
b = \frac{\sigma_{uv}}{\sigma_v^2}, \qquad a = \mu_u - b\,\mu_v,
$$

with $\mu$ the means, $\sigma_v^2$ the variance of $v$ and $\sigma_{uv}$
the covariance of $u$ and $v$. Those, plus the variance of $u$ for the
standard errors later, are everything a record has to hold.

## What Each Pixel Records

### One Pixel's Samples Form a Trapezoid

My first design merged the samples of neighbouring pixels, since one edge's
parallelogram spans two of them. That breaks down as soon as two edges are
less than two pixels apart: more than two consecutive pixels then hold
samples of the same check, and there is no telling which belong to which
edge.

![Four horizontally adjacent pixels. A second edge, with a slightly different slope, sits inside the first pixel, and the original edge sits inside the third pixel. Each edge has its own one-render-pixel-wide, left-neighbor-detectable band — the first edge's band overlaps the first and second pixels, the second edge's band overlaps the third and fourth. All four pixels are labeled "left edge detected"](/assets/images/taa-toon-outline/two-edges-left-detection.svg)

The solution is simple: don't merge. Apply OLS to each rendering pixel
independently, using only that pixel's own jitter samples. Each pixel then
fits a line through the middle of the **trapezoid** formed by cutting the
parallelogram at the pixel boundary, rather than through the middle of the
whole parallelogram.

![Same four-pixel setup as before, but now four red line segments are added, one per pixel, each running through the middle of that pixel's own trapezoid — the piece of its edge's detectable parallelogram left after the vertical pixel-boundary cut. The four red lines are visibly different from each other and from the two black true-edge lines, since each pixel only fit its own half of the data](/assets/images/taa-toon-outline/two-edges-per-pixel-trapezoid-fit.svg)

That line still gives the edge back, with simple algebra. Write the edge in
the pixel's coordinates as $u = E(v) = A + Bv$. The left check fires when
the edge lies between the left neighbour's sample at $u - 1$ and the pixel's
own at $u$, so when $u \in [E(v),\ 1]$ inside the pixel: at each height $v$
the detected samples run from $E(v)$ to 1, and their midpoint is
$(1 + E(v))/2$. The line through all those midpoints is

$$
u = \frac{1 + A + Bv}{2} = \underbrace{\frac{1 + A}{2}}_{a} + \underbrace{\frac{B}{2}}_{b}\,v,
$$

so the fitted $a$ and $b$ decode to the edge as

$$
A = 2a - 1, \qquad B = 2b .
$$

In my first design each pixel then decided from the decoded $A$ whether it
*contained* the edge, and only the containing pixel drew. The plugin does
something else, described under
[Drawing Where the Band Crosses](#drawing-where-the-band-crosses); the
decoding above is used as it is.

### Using Both Checks of an Axis

Recording only the left check's samples throws away information. For the
pixel that contains the edge, the frames on which the left check did *not*
fire are exactly the frames on which the right check did: a sample at
$u < E(v)$ lies on the other side of the edge from the pixel's left
neighbour's sample, but on the same side as its own, and it is the right
neighbour's sample that now sits across the edge. "Not detected by the left
check" and "detected by the right check" describe the same set of samples,
ignoring the boundary itself.

![The same pixel and edge split into the same two trapezoids, labeled region L and region R. A legend reads: region L is right-inducer detected and left-inducer not detected; region R is left-inducer detected and right-inducer not detected — showing the two checks see the same physical split, just named oppositely](/assets/images/taa-toon-outline/opposite-direction-complementary-regions.svg)

The right check's samples fill the complementary trapezoid, $0 \le u < E(v)$,
whose midline is $E(v)/2 = a + bv - \tfrac12$: the same line as the left
check's midline, shifted by exactly half a pixel.

![A single pixel split by a slanted edge into a smaller detected trapezoid on the left and a larger not-detected trapezoid on the right, each with its own OLS midline drawn through its middle; a horizontal double-headed arrow between the two midlines, at the same height, is labeled as exactly a 0.5 pixel shift](/assets/images/taa-toon-outline/not-detected-trapezoid-shift.svg)

So there is no need for two records per axis. The plugin keeps one record per
axis and folds both checks into it: a sample that fired the left (or top)
check is recorded as it is, at $(u, v)$, and a sample that fired the right
(or bottom) check is shifted half a pixel along the axis first and recorded
at $(u + \tfrac12, v)$. Both land on the same midline, $u = \tfrac12 + E(v)/2$,
and the record fits that midline from twice the samples. A frame on which
neither check fires records nothing: the record is left alone, so a pixel far
from any edge stays empty, and the sample count itself tells how often the
pixel sees the edge.

This also means the record uses the full data of both parallelograms next to
the pixel, not half of each. For an edge in pixel B with neighbours A and C,
the horizontal axis has two parallelograms: one straddling the A–B
boundary and one straddling B–C.

![Three square pixels, A, B, and C, with the true edge running through B. A slanted green band, straddling the A–B boundary, covers A's right portion and B's left portion; a slanted purple band, straddling the B–C boundary, covers B's right portion and C's left portion. Vertical pixel-boundary lines cut through both bands](/assets/images/taa-toon-outline/two-parallelograms-cut-by-boundary.svg)

The pixel boundaries cut them into four trapezoids, but only two distinct
sets of samples: A's right check and B's left check are literally the same
comparison between the same two samples, so A's right trapezoid holds the
same frames as B's left trapezoid, and likewise for C and B on the other
side.

![The same three square pixels, A, B, and C. A's right portion and B's right portion are shaded the same blue, on opposite sides of B, labeled as one dataset from A's right-check and B's left-check; B's left portion and C's left portion are shaded the same orange, similarly on opposite sides of B, labeled as one dataset from B's right-check and C's left-check](/assets/images/taa-toon-outline/two-parallelograms-shared-data.svg)

Keep that picture in mind: **one edge's samples live in three pixels**, the
edge pixel with both halves and each neighbour with a copy of one half. It is
what makes the pooling further down work.

### One Record per Axis, Crease or Silhouette

A pixel's axis record is either a crease record or a silhouette record, never
both; the type is stored in the sign of the sample count. A silhouette
sample always takes over a crease record, since a depth discontinuity is the
stronger signal, while a crease sample takes over a silhouette record only
once the silhouette record has nearly faded away. The rules, and why the
takeover threshold has to sit below one sample, are in the
[silhouettes]({% post_url 2026-10-11-thin-outline-silhouettes %}) post.
Everything below treats a record as typeless statistics.

## Storing a Record

Each record holds a decayed sample count $n$, the means $\mu_u, \mu_v$, the
variances $\sigma_u^2, \sigma_v^2$, the covariance $\sigma_{uv}$, the mean
edge strength, and a presence value that the
[flicker suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %})
post explains. That is two RGBA16F texels per axis; with the extra per-axis
data silhouettes need, the plugin stores 44 bytes per rendering pixel, kept
twice over for this frame and the history. Three storage decisions turned out
to matter.

### Central Moments

My first design stored raw running averages, $\langle u\rangle$,
$\langle u^2\rangle$, $\langle uv\rangle$ and so on, and computed the
variance as $\langle u^2\rangle - \langle u\rangle^2$. In fp16 that is a
difference of two nearly equal numbers. Half floats keep 11 significant bits,
and in these records $u$ reaches about 1.5 after the half-pixel shifts, where
$\langle u^2\rangle \approx 2$ and one fp16 step is about $10^{-3}$. Every
frame's store rounds by up to half a step, while the variance being measured
may be $10^{-3}$ or smaller. A simulation of 400 frames of samples through
both schemes, rounding to fp16 after every frame as a texture store would,
with a decay rate of 0.04:

| Band of samples | Exact variance | Raw moments in fp16 | Central moments in fp16 |
|---|---:|---:|---:|
| width 1.0 around u = 0.6 | 0.058672 | 0.058315 (1% off) | 0.058655 (0.0% off) |
| width 0.2 around u = 1.4 | 0.003610 | 0.000266 (**93%** off) | 0.003571 (1.1% off) |
| width 0.05 around u = 1.4 | 0.000218 | −0.000748 (**negative**) | 0.000218 (0.1% off) |

Narrow bands are exactly the slivers next to pixel boundaries, where the
standard errors matter most. So the plugin stores **central moments**, the
mean and the variances and covariance about it, which keep small numbers
small. Central moments also make moving a record trivial: shifting all the
samples by a constant changes only the mean, so the half-pixel shifts above
and the shifts of the history reprojection touch $\mu$ and leave the
moments alone.

### A New Sample Weighs One Over n

The classic decayed average, $\langle v\rangle \leftarrow (1-d)\langle v\rangle + d\,v$,
starts biased toward zero: after one sample it holds $0.04\,v$, and only
about $v/2$ after 17. Every freshly reset record would start wrong, and since
the biases of the different moments differ, its variance would be wrong too.
The plugin instead gives a new sample the weight $\alpha = 1/n$ after
incrementing $n$: the first sample is taken at full value, and from then on
the record is the exact weighted mean of what it has seen. With a sample
every frame, $n$ settles at $1/d$ and $\alpha$ at $d$, the classic update.
The mean and the central moments update together:

$$
\boldsymbol\delta = \mathbf s - \boldsymbol\mu, \qquad
\boldsymbol\mu \leftarrow \boldsymbol\mu + \alpha\,\boldsymbol\delta, \qquad
\Sigma \leftarrow (1-\alpha)\left(\Sigma + \alpha\,\boldsymbol\delta\boldsymbol\delta^\top\right),
$$

where $\mathbf s$ is the new sample and $\Sigma$ the covariance matrix of
$(u, v)$. In the shader, with the record's extra per-sample values updated as
plain weighted means:

```hlsl
// Exponentially weighted update of the running mean and central moments with one new sample of weight 1.
void AddRecordSample(inout FThinOutlineRecord Record, float2 Sample, float Strength)
{
	Record.SampleCount += 1.0;
	const float Alpha = rcp(Record.SampleCount);
	const float2 Delta = Sample - Record.Mean;

	Record.Mean += Alpha * Delta;
	Record.Variance = (1.0 - Alpha) * (Record.Variance + Alpha * Delta * Delta);
	Record.Covariance = (1.0 - Alpha) * (Record.Covariance + Alpha * Delta.x * Delta.y);
	Record.Strength += Alpha * (Strength - Record.Strength);
}
```

### The Count Decays Every Frame

Every frame, with or without a sample, $n \leftarrow (1-d)\,n$. The means and
moments are normalized, so this leaves them unchanged; it only shrinks the
weight of everything recorded so far. The decay rate is 0.04 by default, a
memory of about 25 frames, and a record whose count falls below 0.1 is
cleared.

Decaying on every frame, rather than only on frames that record a sample, is
what makes $n$ mean "how often": a record that gets a sample on a fraction
$f$ of the frames settles at $n \approx f/d$. A pixel on the edge gets a
sample every frame, and its two neighbours get the frames on one side of the
edge each, so their counts add up to the edge pixel's, in the right
proportions. The pooling in the next section weights records by $n$, and
only this decay rule makes that weight the weight of the actual samples. It
also makes stale records fade: an edge that moves away without the history
being rejected decays below the three-sample minimum in about 50 frames and
is cleared after about 135.

## Pooling a Pixel With Its Neighbours

### The Dashed-Line Artifact

With one record per pixel and axis, the pixel containing the edge fits a
line from both halves of the edge's samples, while each neighbour fits a
one-sided copy. When the true edge sits close to a pixel boundary, that
asymmetry showed up as an artifact in my first implementation: statistical
noise could make a pixel conclude it doesn't contain the edge while its
neighbour, working from different data, concluded the same, or the opposite.

![Two pixels, A and B, side by side. A solid black true edge runs very close to their shared boundary, inside A. A dashed green line just to its left represents a reconstructed estimate that correctly lands in A, while a dashed red line just to its right represents a noisy estimate that crosses the boundary and lands in B instead](/assets/images/taa-toon-outline/boundary-flip-noise.svg)

![Three square pixels, A, B, and C, with the true edge inside B. In pixel B, both the left portion (labeled L1, orange) and the right portion (labeled R1, blue) are shaded, since B draws on both datasets. In pixel A, only the right portion is shaded blue and labeled R2, a shifted copy of R1; the rest of A is unshaded. In pixel C, only the left portion is shaded orange and labeled L2, a shifted copy of L1; the rest of C is unshaded](/assets/images/taa-toon-outline/full-vs-half-dataset-access.svg)

Along a mostly horizontal edge, which crosses row after row of pixels near
their boundaries, the contour repeatedly thickened to two pixels or dropped
out entirely: a dashed outline.

![Two panels, each a 6-column, 2-row pixel grid, with the same low-angle black true edge running from the top row on the left to the bottom row on the right. In the 1st approach panel, every column has exactly one red reconstructed segment, though one column near the middle has it in the wrong row. In the 2nd approach panel, one of those middle columns has no reconstructed segment at all, and the next column has two — one in each row](/assets/images/taa-toon-outline/dashed-outline-comparison.svg)

The artifact has two causes. Neighbouring pixels fit **different data**, so
they can disagree about where the edge is. And only the pixel that
**contains** the edge drew, so a disagreement cost a whole segment rather
than a little thickness. The plugin removes both causes: this section the
first, [Drawing Where the Band Crosses](#drawing-where-the-band-crosses) the
second.

### Why Neighbours Can Share Their Records

Neighbouring pixels can fit the same data because, as shown above, the edge
pixel's record is already duplicated piece by piece in its two neighbours: B
holds L1 and R1, A holds a copy of R1, and C holds a copy of L1. The records
are only stored in different coordinates, so a neighbour's record has to be
shifted into the reading pixel's frame first.

![Three square pixels, A, B, and C, drawn slightly apart, with the true edge inside B. B's left portion is shaded orange and labeled L1, its right portion blue and labeled R1. A's right portion is shaded blue and labeled R2; C's left portion is shaded orange and labeled L2. The legend notes that R1 and R2 come from the same frames, and so do L1 and L2](/assets/images/taa-toon-outline/merge-step1-raw-samples.svg)

Each pixel folds its own samples into its record with the rule above: left
check samples as they are, right check samples shifted half a pixel. For B,
R1 is a left-check detection and stays; L1 is a right-check detection and
moves half a pixel right, on top of R1. A's R2 is a right-check detection
from A's point of view, so A shifts it half a pixel right too, past A's own
right boundary. C's L2 is a left-check detection and stays.

![The same three pixels after each pixel's own shift. A's R2 has moved half a pixel right, running past A's right boundary, with its original position shown as a dashed outline. In B, R1 is unchanged and L1 has moved half a pixel right, overlapping R1, with its original position dashed. C's L2 is unchanged](/assets/images/taa-toon-outline/merge-step2-own-shifts.svg)

Now compare. A's right check fires on exactly the same frames as B's left
check, at the same local jitter position, since every pixel shares the
jitter. B kept those samples where they were; A shifted them half a pixel
right. So for every sample they share, A's stored $u$ is half a pixel greater
than B's, and shifting A's record by $-\tfrac12$ lays it on top of R1. C
mirrors this: shifting C's record by $+\tfrac12$ lays it on B's shifted L1.

![The same three pixels as step 2, with two red double-headed arrows labeled 0.5 apart: one between a point in A's shifted R2 and a point in B's R1, and one between a point in B's shifted L1 and a point in C's L2. Each pair holds the same samples, stored half a pixel apart](/assets/images/taa-toon-outline/merge-step3-align-neighbors.svg)

The shift is half a pixel even though the neighbours sit a full pixel away,
because the records don't store raw positions. They store positions folded
into each pixel's trapezoid, whose far side is pinned to the pixel's own
boundary: the midline is $\tfrac12 + E(v)/2$, so moving the edge by a whole
pixel moves the midline by half. The rule from any pixel's point of view: a
neighbour one pixel to the left (or above) is shifted by $-\tfrac12$ along
the axis, a neighbour to the right (or below) by $+\tfrac12$, and nothing
moves across the axis. With central moments the shift is a single
subtraction from the mean:

```hlsl
// Re-expresses a record in the local coordinates of a pixel whose origin is at Offset = (along, across) in the
// record's coordinates. u stores trapezoid midlines, whose far side is pinned to the pixel boundary, so it moves
// at half rate.
void ShiftRecord(inout FThinOutlineRecord Record, float2 Offset)
{
	Record.Mean -= float2(0.5 * Offset.x, Offset.y);
}
```

### Pooling Instead of Fitting the Neighbours Alone

My first implementation drew the strict conclusion from this: each pixel fits
only its two neighbours' shifted records, never its own. Then B fits
A's R2 and C's L2, which are R1 and L1; A fits B's record, which is L1 and
R1; and C fits B's record too. All three fit exactly the same samples and
decode the same line. Pooling a pixel's own record in as well seemed wrong,
because it would make A count R1 twice and C count L1 twice, and the
dashed-line artifact appears exactly where that difference is largest in
relative terms, with the edge a sliver from the boundary.

But "exactly the same data" only holds while all three histories agree. The
moment one pixel's history is rejected by the reprojection, say because it
was just disoccluded, its record restarts with a handful of samples while
its neighbours' are full, and the three pixels fit different data again. My
first implementation guarded this with a reliability test on each neighbour,
falling back to the pixel's own record when a neighbour failed. That guard is
itself a yes-or-no decision, and a freshly reset pixel whose neighbour was
reset too fell back to its own invalid record and drew nothing.

The plugin now **pools each pixel's own record with both neighbours'**,
weighted by their sample counts, with no reliability switch:

![Three square pixels A, B and C in a row with a nearly vertical black edge inside B. A's right portion is shaded blue, B's left portion orange and right portion blue, and C's left portion orange, showing where each pixel's samples come from. Below the pixels, three record boxes labeled with the sample counts n_A, n_B and n_C are joined by arrows into one pool box under B. The arrow from A is labeled "shift −½ along the axis", the arrow from B "as is", and the arrow from C "shift +½". The pool box reads: weighted by n_A, n_B, n_C, then OLS fit, and an arrow from it points back up to the edge in B](/assets/images/thin-outline/pooled-merge.svg)

For a pixel $p$ and the records of the pixels $p + k$ along the axis,
$k \in \{-1, 0, 1\}$, each neighbour's record is first moved into $p$'s
coordinates, $\tilde{\boldsymbol\mu}_k = \boldsymbol\mu_k + (\tfrac k2,\ 0)$
with $\Sigma_k$ unchanged, and then the non-empty records are pooled:

$$
n = \sum_k n_k, \qquad
\boldsymbol\mu = \frac1n \sum_k n_k\,\tilde{\boldsymbol\mu}_k, \qquad
\Sigma = \frac1n \sum_k n_k \left(\Sigma_k + (\tilde{\boldsymbol\mu}_k - \boldsymbol\mu)(\tilde{\boldsymbol\mu}_k - \boldsymbol\mu)^\top\right).
$$

That is the pooled mean and the law of total variance: weighted by $n$, the
pool's statistics are exactly the mean and covariance of the union of the
three records' samples, each sample keeping its decay weight. Any other
weighting gives a covariance that describes no actual set of samples, and
the standard errors built from it lose their meaning. A just-reset record has
a small $n$ and therefore little say, without any threshold deciding when
it starts to count. In the shader the pool is built by merging the records
one after another:

```hlsl
// Statistics of the union of the samples of two records (expressed in the same coordinates).
FThinOutlineRecord MergeRecords(FThinOutlineRecord Record0, FThinOutlineRecord Record1)
{
	FThinOutlineRecord Merged;
	Merged.SampleCount = Record0.SampleCount + Record1.SampleCount;

	const float Weight0 = Merged.SampleCount > 0.0 ? Record0.SampleCount / Merged.SampleCount : 0.0;
	const float Weight1 = 1.0 - Weight0;

	Merged.Mean = Weight0 * Record0.Mean + Weight1 * Record1.Mean;

	const float2 Delta0 = Record0.Mean - Merged.Mean;
	const float2 Delta1 = Record1.Mean - Merged.Mean;
	Merged.Variance = Weight0 * (Record0.Variance + Delta0 * Delta0) + Weight1 * (Record1.Variance + Delta1 * Delta1);
	Merged.Covariance = Weight0 * (Record0.Covariance + Delta0.x * Delta0.y) + Weight1 * (Record1.Covariance + Delta1.x * Delta1.y);
	Merged.Strength = Weight0 * Record0.Strength + Weight1 * Record1.Strength;
	return Merged;
}

// AlongOffset: this pixel's origin along the axis, in the record's coordinates.
void PoolRecord(inout FThinOutlineRecord Pooled, FThinOutlineRecord Record, float AlongOffset)
{
	if (Record.SampleCount > 0.0)
	{
		ShiftRecord(Record, float2(AlongOffset, 0.0));
		Pooled = MergeRecords(Pooled, Record);
	}
}
```

In steady state the three pixels no longer fit exactly the same samples: for
an edge in B, A pools L1 and two copies of R1 while B pools two copies of
each. The pool also counts shared samples twice, so its sample count can
overstate the information by up to a factor of two and the standard errors
come out up to $\sqrt2$ too small. I accepted both because, once every pixel
paints its share of the band instead of only the containing pixel, a small
disagreement between neighbours costs a little thickness rather than a
segment. A synthetic test makes that concrete: give every pixel's fitted
edge position an independent random error of standard deviation $\sigma$,
and count, per column of a straight crease, how often the drawn outline is
under half the true amount (a gap) or over one and a half times it (a
double). For a one-display-pixel outline at 100% screen percentage:

| Position error σ (px) | Containment: gap / double | Band painting: gap / double |
|---:|---|---|
| 0.05 | 3.0% / 0% | **0% / 0%** |
| 0.1 | 5.7% / 0% | **0% / 0%** |
| 0.2 | 8.2% / 0.8% | 0.5% / 1.8% |
| 0.3 | 9.5% / 5.0% | 5.0% / 9.8% |

Small disagreements cost containment whole segments and band painting
nothing. So pixels near an edge still need to fit *nearly* the same data,
but no longer *exactly* the same, and that is what the pool gives. The other
synthetic test resets the history of 20% of the pixels a few frames before
measuring, with the 11-position jitter sequence TSR uses at 100% screen
percentage: the neighbours-only fit with its fallback left gaps in 11.4% of
the columns, the pool in none. On the GPU, with a static camera, a
near-horizontal crease that had come out dashed became nearly continuous,
and diagonal and vertical creases were unchanged.

### What Happened to the Co-Trigger Ratio

The pooling above assumes a single edge crosses the pixel along the axis.
With two edges less than a pixel apart, a sample landing on the sliver
between them fires both checks of the axis in one frame. My first design
kept a decayed count of these **co-trigger** frames per record and used the
ratio as a reliability signal, leaving co-triggered records out of the fit
and treating an axis whose own record was co-triggered as having no edge.

In practice that produced dashed outlines, most visibly in scenes with many
buildings when drawing after the upscaler, so I turned the rule off and then
removed the ratio altogether; its channel now holds the presence. Dense
edges are handled after the fit instead, by the dense edge suppression
described in the [spatial filters]({% post_url 2026-10-11-thin-outline-spatial-filters %})
post.

## The Fit and Its Reliability

### Standard Errors

The pooled record goes into the OLS formula above, and the fit also says how
much to trust itself. The standard errors say how much the slope and the
line's position would wobble if the same edge were sampled again with
different jitter. With $n$ the sample count, the scatter of the samples
around the fitted line, their residual variance, is

$$
s^2 = \frac{n}{n-2}\left(\sigma_u^2 - \frac{\sigma_{uv}^2}{\sigma_v^2}\right),
$$

and the standard error of the slope follows from it:

$$
\mathrm{SE}(b) = \frac{s}{\sqrt{n\,\sigma_v^2}} .
$$

Instead of the intercept's standard error, the plugin computes the standard
error of the line's position at the pixel's centre, $v = \tfrac12$, which is
the least uncertain point of the line and uncorrelated with its slope:

$$
\mathrm{SE}(E(\tfrac12)) = 2\,s\,\sqrt{\frac1n\left(1 + \frac{(\tfrac12 - \mu_v)^2}{\sigma_v^2}\right)},
$$

doubled like the decoded edge. The spatial filter and the dense edge
suppression compare neighbouring pixels through it. A fit is **valid** only
with at least three samples and a spread of $v$ above $10^{-4}$; two points
always fit a line exactly, and with the samples spread over the pixel's
height, a record with less spread has nothing to say about the slope.

### Counting Distinct Jitter Positions

The standard errors assume $n$ independent samples. A temporal jitter
sequence is not independent: TSR cycles through $L$ positions and then
repeats them in the same order. At 100% screen percentage $L = 11$; at 50% it
is 37. That broke the statistics in a way I only found with a GPU dump.

At 100% screen percentage, a diagonal crease in my test level came out
dotted: a second, broken line of dark pixels sat just below it, in every
second row, and at 50% the same crease was clean. The stray pixels had no
record of their own. Each drew the line of its upper neighbour's record, a
sliver record below the edge with $n = 4.94$, a slope of $-2.37$ in its
axis against a true slope of about $+0.55$, and a standard error of
**exactly zero**. The record's statistics had a correlation of $-1.000$:
every sample it held lay on one line, which samples spread over a region can
only do if there are just two distinct ones. Reading the 11 jitter positions
off the sequence, exactly two of them land in that sliver, so every 11 frames
the record added the same two points again. Its decayed count climbed to
about 5, but its information was that of 2 points, and two points fit a line
perfectly, giving the most confident possible verdict about a meaningless
slope.

![A single square pixel with its top edge marked as the boundary to the pixel above. A black line, the true edge decoded from the pixel above, enters at the top near the left and leaves through the right side below the middle, and the light blue sliver between the top edge and this line is the only region where the pixel's top check fires. Eleven small dots mark the jitter positions of TSR's 11-frame sequence spread over the pixel; nine are gray and outside the sliver, and two, in the sliver's right part, are red. A red dashed line runs through the two red dots, far steeper than the black edge and tilted the other way, labeled as the fitted line through the only two recorded positions with zero residual and a standard error of zero](/assets/images/thin-outline/two-point-fit.svg)

The record's data was correct; what was wrong was how much it was trusted.
The fix is to feed the standard errors an estimate of how many *distinct*
jitter positions the record holds, not how many frames. A record that
receives a sample on a fraction $f$ of the frames settles at $n \approx f/d$
and, if its memory of about $1/d$ frames spans at least one cycle, it holds
all $fL$ distinct positions of that fraction, so

$$
n_{\text{distinct}} = n\,d\,L = n \cdot \min(1,\ dL),
$$

where the minimum covers the other case: a memory shorter than a cycle never
sees a position twice, so all its samples are distinct. Nothing extra is
recorded per pixel. The scale is one number per view per frame, with $L$
read from the engine's jitter sequence length, and the shader multiplies the
sample count by it before computing the fit's validity and standard errors.
With the default decay:

| Screen percentage | L | min(1, 0.04 L) |
|---:|---:|---:|
| 100% | 11 | 0.44 |
| 75% | 17 | 0.68 |
| 67% | 19 | 0.76 |
| 50% | 37 | 1.00 |
| 25% | 131 | 1.00 |

Against the dumped records of that frame, the estimate matched the distinct
positions read off the jitter table: the sliver record's $4.94 \times 0.44 = 2.17$
against a true 2, the edge pixel's $24.63 \times 0.44 = 10.84$ against 11,
the pixel above the edge $19.66 \times 0.44 = 8.65$ against 9. Re-running
the composite in Python on the dumped records with the scaled count removed
every stray dot before a line of shader was changed; a sweep over six slopes,
twenty sub-pixel offsets and both sequence lengths found no stray pixel with
the fix anywhere. One effect remains at $L = 11$: lines of one slope sign are
drawn about 0.1 pixels off, because the 11-point Halton set is not
mirror-symmetric; mirroring the jitter moves the error to the other sign. It
is a property of fitting lines to a short, lopsided point set, and it is gone
from 50% screen percentage on.

The whole fit, in the shader:

```hlsl
// DistinctSampleScale converts the decayed sample count into the number of distinct jitter positions behind it, which is
// what the sample count of the standard errors has to be: repeated positions add no information, and a record of only
// two distinct positions fits a line exactly, with a zero standard error.
FThinOutlineEdgeFit FitEdge(FThinOutlineRecord Record, float DistinctSampleScale)
{
	FThinOutlineEdgeFit Fit = (FThinOutlineEdgeFit)0;

	const float SampleCount = Record.SampleCount * DistinctSampleScale;
	Fit.bValid = SampleCount >= THIN_OUTLINE_MIN_FIT_SAMPLE_COUNT && Record.Variance.y > THIN_OUTLINE_MIN_FIT_VARIANCE;
	if (!Fit.bValid)
	{
		return Fit;
	}

	const float B = Record.Covariance / Record.Variance.y;
	const float A = Record.Mean.x - B * Record.Mean.y;

	const float ResidualVariance = max(Record.Variance.x - Record.Covariance * B, 0.0) * SampleCount / (SampleCount - 2.0);
	Fit.SlopeStandardError = sqrt(ResidualVariance / (SampleCount * Record.Variance.y));
	const float CenterOffset = 0.5 - Record.Mean.y;
	Fit.CrossingStandardError = 2.0 * sqrt(ResidualVariance / SampleCount * (1.0 + CenterOffset * CenterOffset / Record.Variance.y));

	// The record fits the trapezoid midline u = 0.5 + E / 2.
	Fit.Intercept = 2.0 * A - 1.0;
	Fit.Slope = 2.0 * B;
	return Fit;
}
```

## Choosing Between the Two Axes

Each pixel reconstructs up to two edges, one from its horizontal axis and
one from its vertical axis, and draws one of them. Silhouette candidates are
considered first, and crease candidates only when neither axis has a
silhouette. Between two candidates of the same type, the slope standard
errors decide: if they differ by more than a threshold (0.05 by default),
the edge with the smaller one is clearly the more stable fit and wins,

$$
\big|\,\mathrm{SE}(b_h) - \mathrm{SE}(b_v)\,\big| > \tau_{\text{SE}}
\;\Longrightarrow\;
\text{pick the smaller } \mathrm{SE}(b) .
$$

Otherwise both are about equally stable and the slope breaks the tie. Each
axis sees an edge best when the edge crosses it squarely, and each edge's
slope is already measured against its own axis, so the better match is the
one with the smaller absolute slope:

$$
\big|\,\mathrm{SE}(b_h) - \mathrm{SE}(b_v)\,\big| \le \tau_{\text{SE}}
\;\Longrightarrow\;
\text{pick the smaller } |b| .
$$

The slope's standard error rather than the position's does the choosing
because an unstable position only slides the line within the pixel, while
an unstable slope means the pixel doesn't know which way the edge runs,
which is a much stronger sign that it is looking through the wrong axis. The
[spatial filters]({% post_url 2026-10-11-thin-outline-spatial-filters %})
post has one more step that runs on the chosen edge before it is drawn.

## Drawing Where the Band Crosses

### Every Pixel Near the Edge Knows the Same Line

My first design had each pixel decide whether it *contains* the reconstructed
edge, and only the containing pixel drew. The decoding
$E(v) = 2(a + bv) - 1$ works just as well for the pixel's neighbours, and
that is what makes a different rule possible. Take one row and the
horizontal axis, with the edge at $u = E$ in a pixel's coordinates, so at
$E + 1$ in its left neighbour's and $E - 1$ in its right neighbour's. Each
pixel's left check records $u$ when the edge lies in $[u - 1, u)$, its right
check $u + \tfrac12$ when the edge lies in $[u, u + 1)$:

| Edge position in this pixel | Left check samples | Right check samples, shifted | Mean of the recorded u |
|---|---|---|---|
| E in [−1, 0], edge in the left neighbour | u in [0, E + 1), mean (E + 1) / 2 | none | 1/2 + E/2 |
| E in [0, 1], edge in this pixel | u in [E, 1], mean (E + 1) / 2 | u in [0, E) + 1/2, mean E/2 + 1/2 | 1/2 + E/2 |
| E in [1, 2], edge in the right neighbour | none | u in [E − 1, 1) + 1/2, mean E/2 + 1/2 | 1/2 + E/2 |

So for every pixel within one pixel of the edge, the expected recorded $u$
at height $v$ is $\tfrac12 + E(v)/2$, with $E(v)$ anywhere in $[-1, 2]$.
When the edge is straight, that is linear in $v$, OLS recovers it, and the
decode gives the edge back: the edge pixel finds $E$, its left neighbour
$E + 1$, its right neighbour $E - 1$. **The same line, each in its own
coordinates.** With the pooled records, the neighbours fit nearly the same
samples too, so in practice they agree closely.

### The Band Test

That makes a purely geometric drawing rule safe: **every pixel paints the
part of the outline band, the edge widened to the outline's thickness, that
lies inside its own square**, whether or not the edge itself passes through
it. Containment is not enough for two reasons. A slanted edge crosses more
than one pixel per row: with the horizontal axis, the edge moves sideways by
the slope over a row's height, and even a slope of 0.5 puts it in two pixels
of the same row for part of their height. And the outline has a thickness:
even a perfectly vertical edge near a pixel boundary has its band spill into
the neighbour, and a one-display-pixel outline at 100% screen percentage is
a full rendering pixel wide, so most edges touch two pixels per row. In both
cases a pixel that doesn't contain the edge has to draw part of it, and it
can, because it knows the same line.

![Two panels, each a grid of five columns and two rows of square pixels with the same black edge running from the top row at the left down into the bottom row at the right, and a gray band around the edge marking the outline's thickness. In the upper panel, labeled containment test, only the pixel in each column whose center line the edge crosses is filled with the band: the band is painted in the top row for the first two columns and in the bottom row for the last three, and where the band spills across the row boundary, at the right end of the second column and the left end of the third, the spilled parts are shown only as a dashed outline, leaving notches in the drawn line. In the lower panel, labeled band test, every pixel is filled with the part of the band inside its own square, and the band is continuous across the grid](/assets/images/thin-outline/band-crossing-rule.svg)

A pixel's sample is always inside its own square, so every piece of the band
is drawn by exactly one pixel, and the line has no gaps or doubled segments.
The band test runs *before* the choice between the two axes: at a corner of
two creases the horizontal edge may cross the pixel while the vertical one
passes just outside, and choosing first could pick the vertical one on its
slope standard error and then draw nothing. Only edges that reach the pixel
compete. A simulation of a straight crease with slope 0.5 at 50% screen
percentage shows what the rule changes; the numbers are the outline alpha
times ten, for the same rows and the same records:

```
center containment                       band test                                true coverage
  7  9  0  0  0  0  0  0  0  0             7  9  4  1  0  0  0  0  0  0             6  9  5  1  0  0  0  0  0  0
  0  0  7  9  0  0  0  0  0  0             0  2  7  9  4  1  0  0  0  0             0  1  6  9  5  1  0  0  0  0
  0  0  0  0  7  9  0  0  0  0             0  0  0  2  7  9  4  1  0  0             0  0  0  1  6  9  5  1  0  0
  0  0  0  0  0  0  7  9  0  0             0  0  0  0  0  2  7  9  4  1             0  0  0  0  0  1  6  9  5  1
```

With containment, each column draws only the pixel whose centre line the
edge crosses, and the band's share in the next row is lost: the line comes
out thinner and uneven, with a staircase of missing pieces. The band test
reproduces the true coverage within about 0.1.

The rule has a price. A containment test is forgiving: an unreliable
neighbour fit whose line only clips a corner of the pixel is ignored, because
its decoded position falls outside. The band test draws whatever a fit says
crosses the pixel, which is exactly how the stray dots above showed up. So
the band test needs fits that are honest about their reliability, which is
what the distinct sample count provides.

## Coverage-Based Alpha

Each check gives its detection a strength, not just a yes or no, and the
record keeps the mean strength of its samples. That strength, times the
band's **coverage** of the pixel, is the alpha by which the pixel's colour is
blended toward the outline colour.

Why coverage, and not a hit test on this frame's sample, comes from how the
upscaler accumulates. My first drawing shader decided each frame's alpha by
point sampling: if the frame's jittered sample fell within the band, the
pixel passed its strength, otherwise zero, and the upscaler was left to
average the hits and misses into a smooth outline. An outline thinner than a
display pixel is hit by only a few jitter positions, so each display pixel
saw it on only a few frames, the history had too few hits to hold on to, and
the outline faded and blinked.

![Three frames of the same render pixel at 50% screen percentage, divided into 2 by 2 display pixels, with a slanted outline band crossing it. In frame 1 the jittered sample lands in the top-left display pixel inside the band, and an arrow leads to a fully black painted result. In frame 2 the sample lands in the same top-left display pixel but outside the band, and the painted result is blank. In frame 3 the sample lands inside the band in the bottom-right display pixel, which the band barely touches, and the painted result is again fully black](/assets/images/taa-toon-outline/coloring-hit-based.svg)

The better rule gives each display pixel the same answer every frame,
whatever the jitter. Given the frame's sample position, find the display
pixel $D$ it falls in and the rendering pixel $R$ it belongs to, and pass

$$
\alpha = s \cdot \frac{\lvert\, \text{band} \cap D \cap R \,\rvert}{\lvert\, D \cap R \,\rvert},
$$

with $s$ the strength: the band's share of the part of the display pixel
that lies inside this rendering pixel. Clipping to $R$ matters when a
display pixel straddles rendering pixels, as it does at 67% screen
percentage. Among the samples landing in $D$, a sample lands in $D \cap R$
with probability $\lvert D \cap R\rvert / \lvert D\rvert$, so the
upscaler's average for $D$ is

$$
\sum_{R} \frac{\lvert D \cap R \rvert}{\lvert D \rvert} \cdot \frac{\lvert \text{band} \cap D \cap R \rvert}{\lvert D \cap R \rvert}
= \frac{\lvert \text{band} \cap D \rvert}{\lvert D \rvert},
$$

the exact coverage of the display pixel, provided all the rendering pixels
overlapping $D$ draw the same line, which is again the property that
neighbours decode the same line.

![Four frames of the same render pixel at 50% screen percentage, divided into 2 by 2 display pixels labeled with their outline coverage: 50% top-left, 0% top-right, 36% bottom-left, 14% bottom-right. In frames 1 and 2 the jittered sample lands in the top-left display pixel, once inside the band and once outside, and both arrows lead to the same 50% gray result. In frame 3 the sample lands inside the band in the bottom-right display pixel and paints a lighter 14% gray. In frame 4 the sample lands in the top-right display pixel, which the band doesn't reach, and paints blank](/assets/images/taa-toon-outline/coloring-coverage-based.svg)

The areas are computed in closed form. In the axis coordinates with the edge
$u = A + Bv$, the part of a rectangle $[u_0, u_1] \times [v_0, v_1]$ on one
side of a line is a one-dimensional integral of a clamped ramp,

$$
\operatorname{area}\{u - Bv \le c\} = \int_{v_0}^{v_1} \operatorname{clamp}\!\big(c + Bv - u_0,\ 0,\ u_1 - u_0\big)\, dv,
$$

which splits into two integrals of $\max(g, 0)$ for a linear $g$, each with
three cases. The band is the difference of two such half planes, at
$c = A \pm t\sqrt{1 + B^2}$ for a half thickness $t$ measured perpendicular
to the edge:

```hlsl
// Integral over Length of max(g, 0), with g linear from G0 to G1.
float IntegrateRamp(float G0, float G1, float Length)
{
	const float GMin = min(G0, G1);
	const float GMax = max(G0, G1);

	if (GMin >= 0.0)
	{
		return Length * 0.5 * (G0 + G1);
	}
	if (GMax <= 0.0)
	{
		return 0.0;
	}
	return Length * 0.5 * GMax * GMax / (GMax - GMin);
}

// Area of {(u, v) in [RectMin, RectMax] : u - Slope * v <= Offset}.
float GetHalfPlaneArea(float Offset, float Slope, float2 RectMin, float2 RectMax)
{
	const float Width = RectMax.x - RectMin.x;
	const float Height = RectMax.y - RectMin.y;

	// Extent along u of the half plane inside the rect, measured from RectMin.x, at both ends of the v range.
	const float G0 = Offset + Slope * RectMin.y - RectMin.x;
	const float G1 = Offset + Slope * RectMax.y - RectMin.x;
	return IntegrateRamp(G0, G1, Height) - IntegrateRamp(G0 - Width, G1 - Width, Height);
}

// Area inside [RectMin, RectMax] of the band of points whose signed distance from the line u = Intercept + Slope * v,
// positive toward +u, lies in [BandMin, BandMax].
float GetBandArea(float Intercept, float Slope, float BandMin, float BandMax, float2 RectMin, float2 RectMax)
{
	const float DistanceToU = sqrt(1.0 + Slope * Slope);
	return GetHalfPlaneArea(Intercept + BandMax * DistanceToU, Slope, RectMin, RectMax)
		- GetHalfPlaneArea(Intercept + BandMin * DistanceToU, Slope, RectMin, RectMax);
}
```

A crease's band is symmetric around the edge, half the thickness on each
side; a silhouette's band lies entirely on the background side, so it wraps
around the foreground object instead of eating into its shape. The thickness
is given in display pixels and converted to rendering pixels by the screen
percentage. A candidate whose band covers less than a ten-thousandth of the
pixel is dropped: a band merely touching the pixel's corner left a rounding
residue that could win the selection with no coverage to show.

This is the rule for drawing before the upscaler. The plugin now draws after
it by default, at display resolution, where every display pixel sums the
band areas of the rendering pixels it overlaps directly and the upscaler
plays no part; the
[drawing and performance]({% post_url 2026-10-11-thin-outline-drawing-and-performance %})
post covers that mode, the comparison between the two, and the tilt the
after-upscaler mode made visible.

## What the Other Posts Cover

Everything above treats a record as if it stayed put. The
[history reprojection]({% post_url 2026-10-11-thin-outline-history-reprojection %})
post covers how records follow their surfaces when the camera or the
objects move, and when they are thrown away. The
[silhouettes]({% post_url 2026-10-11-thin-outline-silhouettes %}) post covers
how depth edges differ: which pixel records them, where their band is drawn,
and why the upscaler needs the foreground's depth and velocity under the
outline. The [spatial filters]({% post_url 2026-10-11-thin-outline-spatial-filters %})
post covers what is blended between neighbouring pixels and which edges are
suppressed, and the
[flicker suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %})
post covers what still blinked once all of this was in place, and why.
