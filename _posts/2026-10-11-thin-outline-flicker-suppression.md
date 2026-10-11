---
layout: post
title: "Thin Outlines From the G-Buffer Under TAA: Flicker Suppression"
date: 2026-10-11 14:00:00 +0900
categories: [rendering, npr]
hidden: true
---

Drawn after the upscaler, the outline no longer fades or breaks into dashes,
but on some surfaces at some distances it flickers: a line that is there on
one frame and gone on the next. This post covers what flickers and why, how
I measured it over the jitter cycle, the two suppressions I tried first and
why they did not help, and what did: the presence of a record, a filter for
normal spikes at sampling time, and a filter for foregrounds one pixel thin.
It is part of a [series]({% post_url 2026-10-11-thin-outline-plugin %}) and
assumes the [core idea]({% post_url 2026-10-11-thin-outline-core-idea %}),
[history reprojection]({% post_url 2026-10-11-thin-outline-history-reprojection %})
and [spatial filters]({% post_url 2026-10-11-thin-outline-spatial-filters %})
posts.

*The sections on what flickers, the measurements, the presence, the spike
filter, the thin-foreground filter and the stabilization that was built and
removed are not written yet. The two suppressions below are complete.*

## Dense Edge Suppression

### The Problem

When many edges crowd into a small area of the screen, the outline flickers.
I saw it on a rolling shutter (dense silhouettes) and on the tiles of a
building (dense creases), each from a certain distance. The cause is that
once edges sit only a pixel or two apart along a pixel's axis, the jitter
decides which of them each pixel's checks pick up on a given frame. The
records of the pixels between them mix the samples of several edges, and the
line fitted from a mixed record wanders from frame to frame. Drawn before the
upscaler, the upscaler received an outline that never settled and blurred
it; drawn after the upscaler, the wandering is visible as it is.

My first rule, for a pixel with a horizontal-axis edge, looked at the two
pixels to its left and the two to its right and suppressed the edge when
both distance-1 neighbours also contained an edge of the same axis, since
three in a row cannot be one thin edge, or when either distance-2 neighbour
did, since two edges whose two-pixel-wide detection bands overlap mix their
samples in the records between them. One distance-1 neighbour alone was not
dense: a single edge near the shared boundary shows up in both pixels. The
rule in the plugin keeps the reach and changes how the edges are counted,
and it starts from one fact about the records.

### Where One Edge's Records Are

Take a vertical edge at $E = p + e$ inside pixel $p$, with $e \in [0, 1)$,
and the jitter $j \in [0, 1)$ along the axis. Pixel $p$ gets samples from
both of its checks on every frame; $p - 1$ gets right-check samples on the
frames with $j > e$, and $p + 1$ left-check samples on the frames with
$j < e$. Three pixels hold records of one edge. And since a record stores
the trapezoid midline $u = \tfrac12 + E/2$ in its own pixel's frame, its mean
$\mu_u$ decodes to the edge's position $2\mu_u - 1$ at the record's mean
$v$ exactly, because the fitted line passes through the mean point. No slope
is needed to ask *where* a record's edge is. For $p$'s record that is $e$,
inside $p$; for $p - 1$'s it is $1 + e$ and for $p + 1$'s it is $e - 1$,
both **outside** those pixels, on $p$'s side:

![Two rows of five square pixels along the horizontal axis. In the upper row, labelled one edge, a thick black vertical edge stands inside the middle pixel p at 0.7 of its width, and record boxes under p minus 1, p and p plus 1 read right-check samples, both checks and left-check samples; dashed red arrows from each box point at the edge's foot, and the labels say the records decode to 1.7, outside, to 0.7, inside, and to minus 0.3, outside, so only p contains the edge. In the lower row, labelled two edges two pixels apart, edges E1 and E2 stand at 0.7 inside pixels p and p plus 2, and record boxes under the five pixels read E1 right, E1 both, E1 left plus E2 right, E2 both and E2 left; the verdicts under them read drawn, drawn because it contains E1, mixed and suppressed, drawn because it contains E2, and nothing to draw, and the caption explains that p plus 1 sees an edge inside the pixel on each side whose fit crosses elsewhere than its own. A footer notes that a record decodes to the edge position two mu u minus one at its mean v, with no slope needed](/assets/images/thin-outline/dense-edge-records.svg)

So a record in a pixel does not mean an edge in that pixel. A pixel next to
an edge holds that edge's record, and the record decodes to a position
outside the pixel. "The pixel contains an edge" is a test on the record's
mean:

```hlsl
// Whether a record's edge lies inside its pixel: the midline mean decodes to the edge's position at the record's mean v
// (the fitted line passes through the mean point).
bool IsRecordEdgeInside(FThinOutlineRecord Record)
{
	const float Edge = 2.0 * Record.Mean.x - 1.0;
	return Record.SampleCount > 0.0 && Edge >= 0.0 && Edge < 1.0;
}
```

### Counting the Edges on a Side

Along the drawn edge's axis, a side (left or right; top or bottom for the
vertical axis) holds **another edge** when any of these holds, tested
cheapest first:

1. **Distance 2: the pixel contains an edge.** Its own record, of either
   type, decodes to a position inside it. It cannot be this pixel's edge:
   the selected axis has $\lvert b \rvert \le 1$, so one edge cannot lie
   inside two pixels two apart.
2. **Distance 1: a record of another kind.** The neighbour's own record is
   of the other type, or a silhouette record of the other background side.
   It cannot be this pixel's edge.
3. **Distance 1: a fit of the same kind that puts the edge elsewhere.** Only
   when the neighbour's own record decodes to a position inside the
   neighbour; otherwise it is this pixel's edge seen from that side, or the
   edge two steps away, which the first test covers. The neighbour's
   **pooled fit** of the drawn kind, what that pixel would draw, has its
   centre crossing compared with this pixel's, in this pixel's coordinates
   (the neighbour's plus or minus 1).

In the third test the two count as **one edge** when

$$
\lvert u_q \pm 1 - u_p \rvert \le t \sqrt{\mathrm{SE}(u_p)^2 + \mathrm{SE}(u_q)^2},
\qquad t = 2 ,
$$

with the standard errors those of the centre crossings, the quantity the
[spatial filter]({% post_url 2026-10-11-thin-outline-spatial-filters %})
weighs by. Within the tolerance the two pixels have
reconstructed the same edge with some error; outside it, two edges.

```hlsl
const float NeighbourCrossing = NeighbourFit.Intercept + 0.5 * NeighbourFit.Slope + float(Side);
const float DifferenceVariance = OwnCrossingVariance + Pow2(max(NeighbourFit.CrossingStandardError, THIN_OUTLINE_MIN_STANDARD_ERROR));
return Pow2(NeighbourCrossing - OwnCrossing) > Pow2(DenseEdgeTolerance) * DifferenceVariance;
```

The third test replaces my first rule's "both distance-1 neighbours": a
neighbour that draws the same edge is not evidence of density, however many
of them there are. The containment test of the first rule came from a wrong
first version, which counted *any* record two steps away. A pixel drawing
the band near its boundary then counted its own edge's record, two steps
away in the next pixel but one, as another edge; with the either-side trigger
below, that version removed 56% of the outline alpha of my test scene, and
with the containment test 17%.

### The Trigger

The edge is suppressed when **both** sides hold another edge. That is
stricter than my first rule, which suppressed on a distance-2 edge on one
side alone; the either-side trigger is kept as an option. What the two do to
the simplest cases, parallel vertical edges with the records as the checks
lay them down:

| Case | Records | Both sides | Either side |
|---|---|---|---|
| An isolated edge in $p$, its band also crossing $p - 1$ | $p - 1$ (outside), $p$ (inside), $p + 1$ (outside) | $p$ and $p - 1$ drawn: $p - 1$ sees $p$'s record inside $p$, same kind, same crossing; $p + 1$'s record decodes outside $p + 1$ | the same |
| Two edges 1 px apart, in $p$ and $p + 1$ | $p - 1$ to $p + 2$, with $p$ and $p + 1$ mixed | nothing suppressed: $p + 2$'s record decodes outside $p + 2$, and the two mixed pools agree within their inflated errors | nothing |
| Two edges 2 px apart, in $p$ and $p + 2$ | $p - 1$ to $p + 3$, with $p + 1$ mixed | $p + 1$ suppressed, $p$ and $p + 2$ kept | $p$, $p + 1$ and $p + 2$ suppressed |
| Three or more edges 1 to 2 px apart | all | the inner pixels suppressed | all suppressed |

The both-sides trigger keeps an isolated pair of lines and hides the mixed
pixel between them when there is one; it removes lines only where three or
more edges crowd. Geometry one pixel wide, a foreground strip or a thin
spike whose two edges are a single pixel apart, is an isolated pair and
slips through, which is why its samples are filtered at sampling time
instead, further down in this post. The narrow background gap between two
foreground parts of the
[silhouettes]({% post_url 2026-10-11-thin-outline-silhouettes %}) post is the
same case for silhouettes: the gap pixel's neighbour on the far side holds
a silhouette record of the other background side, which the second test
counts, but the near side holds the gap's own contour, so the both-sides
trigger leaves the gap alone and only the either-side trigger removes it,
with much else.

### What It Removes and Costs

In my test scene, static at 1280×720 and 73% screen percentage, drawn after
the upscaler, the both-sides trigger removes 7.1% of the outline alpha
(2,241 display pixels touched, 19,334 drawn) and the either-side trigger
16.7% (5,030 touched). The removed pixels under the both-sides trigger are
the dense creases on the character's torso and hands, a few junctions where
ramp edges meet, and the inner of two parallel lines along a pillar's edge;
every silhouette stays.

| Drawing mode | Off | Both-sides trigger |
|---|---|---|
| after the upscaler, 73% | 0.139 ms | 0.147 ms (+6%) |
| before the upscaler, 100% | 0.141 ms | 0.147 ms (+4%) |

The tests run cheapest first, and the second side only when the first
leaves the rule open: for an isolated edge, no pooled fit is computed at
all. A first version that computed both sides' pooled fits for every drawn
pixel cost +14% at 73%; the cost was the fits and the record copies, not a
load round trip.

What the suppression is not: a fix for the flicker. In Epic's City Sample,
where the window frames and facade tiles at a distance flicker, it did not
help, because a mixed record's wandering is only one of the things that
flicker and not the main one. I kept it anyway: where edges are dense it
stops the spot from being darkened by a tangle of lines, which a user may
or may not want, and it is a switch.

## Isolated Edge Suppression

The opposite case is an edge that is too *small*. An edge of one or two
pixels is a speck: a record that comes and goes with the jitter rather
than a line, and it flickers. The rule is that a drawn edge must run over
at least three pixels across its axis, this one included: it continues into
both neighbours across the axis, or into one and the pixel beyond it, which
is an edge end or a corner. "Continues" means that the pooled record of the
drawn kind at that pixel, pooled as that pixel pools it, can be fitted, with
the fit's minimum of three distinct samples and some spread across the axis.
These are the two neighbour fits of the
[spatial filter]({% post_url 2026-10-11-thin-outline-spatial-filters %}), so
the test starts from them and loads one more pooled record only when
exactly one neighbour continues:

```hlsl
// At least three pixels in a row across the axis: both neighbours, or one and the pixel beyond it.
bContinues = Before.bValid && After.bValid;
BRANCH
if (bIsolatedEdgeSuppression != 0 && Before.bValid != After.bValid)
{
	const int Beyond = Before.bValid ? -2 : 2;
	bContinues = CanFitEdge(GetAcrossRecord(bSilhouette, Candidate.bBackgroundAlongU, Axis, ViewportPos, Beyond), DistinctSampleScale);
}
```

In the test scene at 73% after the upscaler it removes 0.6% of the outline
alpha, 355 display pixels, and costs 0.002 ms on top of the filter (0.0005
ms before the upscaler at 100%). I built it first as a flicker remedy for
the City Sample, found that removing one- and two-pixel edges did nothing
for the flicker there, and took it out. It came back once the presence and
the spike filter below were in: some surfaces with grooves
still flickered at certain distances, the presence ramp could not be tuned
for them without making other creases at the same distance dashed, and the
specks were the handle that was left. Sharing its loads with the filter
made it nearly free, which is the other reason it stayed.

## What the Two Suppressions Are Not

Both run once per rendering pixel after the spatial filter, on the edge the
axis selection chose and before the band's coverage is computed: the dense
test first, since it is cheaper and decides more pixels. Neither fixed the
flicker, and that was what each was tried against first.

| Step | What it fixes | What it does not |
|---|---|---|
| Dense edge suppression | Tangles of lines where three or more edges crowd within two pixels | The flicker of dense edges; isolated pairs, including one-pixel geometry |
| Isolated edge suppression | Specks of one or two pixels | The flicker of lines; it removes 0.6% of the outline |
