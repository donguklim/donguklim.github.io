---
layout: post
title: "Thin Outlines From the G-Buffer Under TAA: History Reprojection"
date: 2026-10-11 12:00:00 +0900
categories: [rendering, npr]
hidden: true
---

An edge record is built up over many frames, so when the camera or the
objects move, each record has to follow the surface it describes, the way a
temporal upscaler's history buffer does. This post covers how a record is
fetched from the previous frame and moved into its new pixel, and the tests
that decide when a record no longer describes the surface under its pixel: a
depth test for creases, two tests for silhouettes, and a velocity test for
both. It is part of a [series]({% post_url 2026-10-11-thin-outline-plugin %})
and assumes the [core idea]({% post_url 2026-10-11-thin-outline-core-idea %})
and the [silhouettes]({% post_url 2026-10-11-thin-outline-silhouettes %})
post.

## Moving a Record to Its New Pixel

Each frame, before this frame's samples are added, every rendering pixel
fetches the previous frame's records at its reprojected position: the
pixel's centre moved back by its motion vector. A pixel of a moving object
reads its motion vector from the velocity buffer. Static geometry draws no
velocity, so its motion is the camera's alone, computed from the pixel's
depth and the previous and current projections, as the upscaler does it.
Both projections are taken without their jitter, so the reprojection is
between pixel centres, not between the jittered sample positions.

The reprojected position rarely lands on a pixel centre, and a record's
statistics are expressed in its own pixel's local coordinates, so the record
has to be moved into the current pixel's. This is the half-rate shift of the
core idea post, for an arbitrary offset. If the current pixel's centre lies
at $(o_u, o_v)$ from the history pixel's centre, along and across the axis,
the record's mean moves by

$$
\mu_u \leftarrow \mu_u - \tfrac12 o_u, \qquad \mu_v \leftarrow \mu_v - o_v ,
$$

and the variances and covariance stay as they are. The half rate is the same
as for the neighbour pooling: the record stores trapezoid midlines pinned to
the pixel's own boundary, so an edge that moves by $o_u$ moves the midline by
$o_u/2$. My first design stored raw moments, and a shift had to update five
running averages, each with its cross terms; with central moments the whole
move is the two subtractions above, in the `ShiftRecord` function the core
idea post quotes.

## Nearest Pixel or Bilinear Taps

### Records Slip Past a Moving Edge

My first implementation fetched the record of the history pixel nearest to
the reprojected position and shifted it by the remaining sub-pixel offset.
That is geometrically right: the shifted record still describes the edge at
its true position. But the record itself only moves by whole pixels, while
the edge moves by a fraction of a pixel every frame. Record and edge slip
apart by the velocity's distance to the nearest integer, every frame: with a
fraction below half a pixel per frame the record stays in its pixel while the
edge moves away, so records trail behind the edge; above it, the record
jumps a whole pixel each frame and runs ahead.

A pixel can only sample an edge that lies in it or in one of its two
neighbours along the axis. Once a record has slipped out of that range it
never gets another sample, and it lingers while its count decays by 4% per
frame:

![Three rows of five square pixels labelled A to E, one row per frame, with a nearly vertical black edge that moves 0.3 pixels to the right per frame, and a box labelled "B's record" under pixel B in every row. In frame 0 the edge is inside B, and a blue bracket under A, B and C marks the edges B can sample. In frame 3 the edge has moved into C; a short gray arrow inside B shows B's centre traced back 0.3 pixels, landing in B, so B fetches its own record, and a red dashed arrow from the record box to the edge shows that the record still decodes to it. In frame 6 the edge is inside D, outside the bracket, which is labelled in red as out of B's range; the red dashed arrow still reaches the edge, and the caption says the record never gets another sample and lingers while its count decays by 4% per frame](/assets/images/thin-outline/record-slip.svg)

I found this as a smear: with the camera moving, the records' sample counts,
viewed as an image, trailed or led every moving crease by a band 20 to 30
pixels wide that faded out over a second. In a frame captured under a steady
camera pan at 1280×720 and 100% screen percentage, the horizontal-axis crease
records along one row of a vertical crease looked like this:

| x | n | Mean u | Edge the record decodes to |
|---:|---:|---:|---:|
| 430 | 5.43 | 0.835 | (fit invalid) |
| 431 | 9.27 | 0.318 | 430.65 |
| 432 | 7.95 | −0.215 | 430.59 |
| 433 | 5.96 | −0.771 | 430.5 |
| 436 | 2.93 | −2.715 | 430.1 |
| 444 | 0.15 | −8.289 | 430.2 |

Every record still points at the edge at $x \approx 430.5$, from up to 14
pixels away, and the pixels at the edge hold only 5 to 9 samples instead of
the 25 a static edge accumulates, because records keep sliding through the
three-pixel range in which they can be fed.

### Four Taps, Weighted by Sample Count

So the history is fetched the way a temporal upscaler resamples its own:
from the four history pixels around the reprojected position, with bilinear
weights. Each tap's record is moved into the current pixel's coordinates by
its own offset, its sample count is multiplied by its weight, and the four
records are merged with the pooling of the core idea post:

![A three-by-three grid of last frame's pixels with their centres marked, and a red dashed square, this pixel's square traced back by its velocity, whose red centre lands between the centres of the four pixels in the upper right of the grid. A blue dashed rectangle connects those four centres, with the fractions f x equals 0.2 and f y equals 0.9 marked along its top and left sides, and the four pixels carry their bilinear weights 0.08, 0.02, 0.72 and 0.18. A table on the right lists, for the horizontal axis, each tap's weight, its offset, this pixel's centre relative to the tap's centre, and the shift of the record's mean, half the offset along the axis and the full offset across; a note says the merged count is 0.08 times the first tap's count plus 0.02, 0.72 and 0.18 times the others'](/assets/images/thin-outline/bilinear-taps.svg)

With $f_x$ and $f_y$ the fractional position of the reprojected centre
between the four pixel centres, the weights are the usual
$(1 - f_x)(1 - f_y)$, $f_x(1 - f_y)$, $(1 - f_x)f_y$ and $f_x f_y$. Scaling
a record's sample count by its weight scales the weight of every sample in
it, so the merged record holds the statistics of the union of the four
records' samples, each tap's samples at that tap's weight, and the merged
count is the bilinear interpolation of the four counts:

```hlsl
// Merges one history record, moved into this pixel's coordinates, into the reprojected record with a weight: its
// samples keep their relative weights, scaled by Weight.
void AddHistoryRecord(inout FThinOutlineRecord Reprojected, FThinOutlineRecord Record, float2 Offset, float Weight)
{
	ShiftRecord(Record, Offset);
	Record.SampleCount *= Weight;
	Reprojected = MergeRecords(Reprojected, Record);
}
```

Records then follow the edge continuously, at the price of four history
loads per pixel instead of one. A tap outside the view gets no weight, and
the weights of the others are not renormalized: a record that came half from
outside the view counts as half a record. Each tap passes the history tests
below on its own, so a pixel at a boundary between two surfaces takes only
the taps of its own surface. With a static camera the reprojected position
is the pixel's own centre, one tap has all the weight, and nothing changes
against the nearest fetch.

Bilinear taps have a cost in sample counts, too: every frame the edge
pixel's record is blended with its neighbour's, which holds fewer samples,
so under motion the counts stay below the static count. Under the pan above,
the near-horizontal creases moving along themselves drew about 20% more
outline than with the nearest fetch, and the smear was gone. Something else
appeared instead: the moving vertical creases widened to a few pixels, the
bilinear blur, which the next section removes.

## Dropping Records Out of Sampling Range

The core idea post's table of recorded $u$ against the edge position says
where a pixel's samples can lie: midlines $\tfrac12 + E/2$ for an edge at
$u = E \in (-1, 2)$, so $u \in [0, 1.5]$. After the reprojection, a crease
record whose midline lies outside that range over the pixel's whole height,
$v \in [0, 1]$, describes an edge the pixel can no longer sample, and it is
dropped instead of lingering. The midline has to follow the record's own
slope: a slanted edge that moves *across* the axis keeps its mean $u$ while
its samples leave the pixel, and a first version that tested only the mean
left the diagonal creases smeared.

```hlsl
const float Slope = Record.Variance.y > THIN_OUTLINE_MIN_FIT_VARIANCE ? Record.Covariance / Record.Variance.y : 0.0;
const float MidlineAtV0 = Record.Mean.x - Slope * Record.Mean.y;
const float MidlineAtV1 = MidlineAtV0 + Slope;
if (max(MidlineAtV0, MidlineAtV1) < 0.0 || min(MidlineAtV0, MidlineAtV1) > 1.5)
{
	Record = GetEmptyRecord();
}
```

The four combinations, in the same captured frame as above (a steady
camera pan, 1280×720, 100% screen percentage, drawn before the upscaler).
"Stranded" records are those whose midline lies outside the sampling range;
the alpha columns are the summed outline alpha drawn over the frame by each
record type and axis:

| Fetch | Horizontal-axis crease records, stranded | Alpha: creases, horizontal axis | vertical axis | silhouettes, horizontal axis | vertical axis |
|---|---|---:|---:|---:|---:|
| nearest | 43,066, 80% | 620 | 4,584 | 37 | 2,378 |
| nearest, range drop | 8,549, 0% | 715 | 4,490 | 22 | 2,284 |
| bilinear | 33,298, 36% | 545 | 5,541 | 41 | 2,746 |
| bilinear, range drop | 20,410, 0% | 568 | 5,516 | 32 | 2,742 |

The pan moves the near-vertical creases, those of the horizontal axis,
across their axis, which is where the slip shows. With the nearest fetch,
four records in five are stranded, and part of what they draw is stray dots:
beside one sloped silhouette, dots of alpha 0.6 on every other column, which
is most of the silhouette alpha the range drop removes. Bilinear taps with
the range drop are the combination I kept: thin lines on the vertical and
diagonal creases alike, no band of stranded records, and the most outline on
the edges moving along themselves.

### Why Silhouette Records Have No Range Drop

The range drop applies to crease records only. A silhouette's records are
kept by the foreground pixels, one and two pixels from the contour on the
foreground side, at the far end of the sampling range, and the bilinear
history spreads a moving contour's records over one more pixel each frame.
The range drop then removed the share that landed one pixel further inside
the foreground, although the contour was right next to it: under the pan,
the character's horizontal silhouette records lost a quarter of their
samples and the drawn silhouette about a tenth. How far a silhouette record
may drift is bounded by the background test below instead.

## Which Velocity Moves the Records

Every pixel fetches its history once, with its own velocity. Both record
types belong to the surface under the pixel: creases by nature, and
silhouettes because the foreground pixels keep them, as the silhouettes post
explains. The one exception is the pixel at a contour on the frames when its
sample lands on the background. Its records come from the other frames, when
it sampled the foreground, so on these frames it fetches with the velocity
of the neighbour its background check fired toward, the nearest one if
several fired:

```hlsl
FThinOutlineForeground FindHistoryForeground(int2 PixelPos, FThinOutlineSample Center, FThinOutlineSample Neighbours[4], FThinOutlineEdges Edges)
{
	FThinOutlineForeground Foreground;
	Foreground.PixelPos = PixelPos;
	Foreground.DeviceZ = Center.DeviceZ;
	Foreground.bFound = false;

	// Reversed Z: nearer is larger.
	UNROLL
	for (uint Index = 0; Index < 4; ++Index)
	{
		if (Edges.BackgroundSilhouette[Index] > 0.0 && (!Foreground.bFound || Neighbours[Index].DeviceZ > Foreground.DeviceZ))
		{
			Foreground.PixelPos = ClampToViewRect(PixelPos + NeighbourOffsets[Index]);
			Foreground.DeviceZ = Neighbours[Index].DeviceZ;
			Foreground.bFound = true;
		}
	}

	Foreground.Depth = DeviceZToDepth(Foreground.DeviceZ);
	return Foreground;
}
```

The crease records of such a pixel ride on the same fetch; only the crease
depth test below keeps using the pixel's own depth change. Each tap's
records feed the reprojected record of their own type, crease or silhouette,
and the axis then keeps one of the two by the takeover rules of the
silhouettes post. When silhouette records were still kept on the background
side, every pixel had to search for the foreground up to two pixels away
and fetch the history a second time with its velocity; moving the records
to the foreground is what made one fetch with the pixel's own velocity
possible.

## The Crease Depth Test

As in TAA, history is thrown out when the surface under a pixel changes,
and depth is the first thing to compare. Each pixel stores the linear depth
of its surface with its records. A crease tap is kept when its stored depth
matches the pixel's own depth moved to the previous frame, by the depth
component of its motion vector, within 5%:

$$
\lvert\, D_{\text{tap}} - D_{\text{prev}} \,\rvert \le 0.05\, D_{\text{prev}} .
$$

The pixel alone is enough. A crease lies on a depth-continuous surface, so
a history pixel of the same surface differs from the pixel only by the depth
change across the jitter: the two samples being compared are up to a pixel
apart on the surface. That is also why the threshold is looser than a
colour TAA's. On a surface seen at a grazing angle a pixel's step in depth
is large, it doubles at 50% screen percentage, and a crease is where the
surface bends, so the depth around it does not follow a single plane. The
rejection rates in a static view at 1280×720, over every pixel whose history
holds a crease record:

| Threshold | Rejected at 100% | at 50% |
|---:|---:|---:|
| 2% | 0.66% | 14.6% |
| 3% | 0.64% | 1.84% |
| **5%** | 0.57% | 1.76% |
| 10% | 0.53% | 1.68% |

At 50%, pixels whose depth changes by 2 to 10% toward a neighbour failed 38
to 47% of their frames at 2%, and about 0.5% at 5%. What remains at any
threshold are pixels on the character's outline whose sample landed on the
character in one frame and on the wall behind it in the other, at depths of
400 and 2,070. Their crease record belongs to a surface the pixel no longer
shows, so rejecting it is right.

An earlier version accepted a stored depth anywhere in the depth range of
the pixel's same-surface neighbours, widened by 2%. It rejected almost
nothing, but it cost four edge measures per pixel, and it kept exactly those
records: a sample that had crossed a depth step still found a neighbour on
the old surface. Under a pan it kept a diagonal crease band 10 to 18 pixels
wide around a depth step where the pixel test keeps 2 to 10.

What a depth test cannot see is a crease record that slid along one
continuous surface: the records a foot leaves on the floor it stood on, a
junction's crease carried onto the curved wall next to it as the camera
uncovers the wall, or the copies a face widening on screen gets from the
crease beside it, since the history only shifts records. The first is caught
by the velocity test at the end of this post; the other two by the presence
of the [flicker suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %})
post, which notices that no crease is found at the pixel any more.

## The Silhouette Tests

A silhouette record carries two more values than a crease record, both
introduced in the silhouettes post: the kept ratio, which says on which side
of the axis the record's background lies, and the foreground depth, a
running mean of the recording pixel's own depth, which is the depth of the
object in front. The stored foreground depth is last frame's, so before the
tests it is moved by the depth change of the pixel whose velocity moved the
history.

### The Foreground in Place

A silhouette tap is kept when its foreground depth matches, within 5%, the
depth of the pixel itself or of its neighbour toward the record's foreground
side. The neighbour has to count because the pixel at the contour samples
the background on some frames, and its foreground is then next to it:

```hlsl
// Whether a reprojected silhouette record's foreground is still in place: the pixel itself, or its neighbour toward
// the record's foreground side, has the record's foreground depth. The pixel at the contour samples the background on
// some frames, and its foreground is then the neighbour. The neighbour depths are those of the first (left or top) and
// second (right or bottom) neighbour of the axis.
bool IsForegroundInPlace(FThinOutlineRecord Record, float CenterDepth, float FirstNeighbourDepth, float SecondNeighbourDepth)
{
	// Samples recorded through the first check have their background on the first side.
	const float NeighbourDepth = Record.KeptRatio >= 0.5 ? SecondNeighbourDepth : FirstNeighbourDepth;
	const float Tolerance = SilhouetteHistoryDepthThreshold * Record.ForegroundDepth;
	return any(abs(float2(CenterDepth, NeighbourDepth) - Record.ForegroundDepth) <= Tolerance);
}
```

With the records kept on the background side, as in my first layout, the
test had to reach two pixels toward the foreground: the pixel one outside
the contour sees its foreground two pixels away on the frames when the
contour pixel samples the background. In a static view, with records on the
background side and a 2% tolerance:

| Pixels compared with the foreground depth | Rejected at 100% | at 50% |
|---|---:|---:|
| the pixel and one step | 35% | 40% |
| the pixel, one and two steps | 2.7% | 3.5% |

The remaining few percent did not depend on the tolerance; they were records
whose foreground really had moved, such as the animated character's head
against the sky. With the records on the foreground, one step is enough.
The 5% tolerance comes from the same place as the crease threshold: a
foreground seen at a grazing angle changes depth by a few percent per row,
and the jitter moves both samples by up to a pixel. Along the grazing top of
a ramp in my test level, 2% lost a part of the silhouette under a pan that
5% kept. The foreground depth is stored in half precision, and a running
mean stored that way stalls when an update is smaller than half a float
step; a simulation of 400 frames with samples spread by up to 3% kept the
stored mean within 0.4% of the exact one, well inside the tolerance.

### Background Within Reach

A silhouette is view-dependent: as the view direction turns relative to the
surface, the contour slides over it. A record that moved with its foreground
surface can therefore end up inside the foreground, where its contour no
longer is, and the foreground test can't tell, because the foreground is
still there. So, toward the record's background side, there has to be a
background within two pixels. A background is within reach in a direction
when any of these holds:

- the pixel itself samples the background of a silhouette this frame, that
  is, its background check toward the opposite side fires;
- the neighbour on that side is behind a silhouette, so the pixel's own
  silhouette check toward it fires;
- the pixel beyond that neighbour is the background: it is farther than the
  neighbour and off the line through the pixel and the neighbour, the
  silhouette measure seen from the neighbour without its symmetric term,
  which would need a third pixel. A plain depth step of 5% in its place gives
  nearly the same picture.

Two pixels, because the foreground pixel one inside the contour has no
background neighbour on the frames when the contour pixel samples the
foreground: the background is then two pixels away. The four directions are
tested once per frame, before the fetch, and each reprojected silhouette
record looks up its own background side:

```hlsl
// Whether a reprojected silhouette record of the axis whose first neighbour is FirstIndex (NeighbourOffsets order) has
// background within reach toward its background side (see GetBackgroundInReachMask).
bool HasBackgroundInReach(FThinOutlineRecord Record, uint FirstIndex, uint BackgroundInReachMask)
{
	// Samples recorded through the first check have their background on the first side.
	const uint BackgroundIndex = Record.KeptRatio >= 0.5 ? FirstIndex : FirstIndex + 1;
	return (BackgroundInReachMask & (1u << BackgroundIndex)) != 0;
}
```

This test runs on the merged record, after the taps, since the background
side is a property of the record, not of a tap.

### The View-Direction Test, Tried and Shelved

Before the background test, the view-dependence of silhouettes was handled
the way my first design proposed: reject a silhouette record when the view
direction through the pixel has turned by more than a threshold angle since
the record was made. The test went through two refinements, and I describe
them because they show why a test on the view direction was the wrong
tool.

The first refinement was to measure the turn in the right plane. At a
smooth contour, the plane through the camera that contains the edge on
screen is the surface's tangent plane. Turning the view direction within that
plane moves the contour along itself; turning it out of the plane slides the
contour over the surface. So only the angle between last frame's view
direction and the current edge plane counted:

$$
\mathbf n = \frac{\mathbf r \times \Delta\mathbf r}{\lvert \mathbf r \times \Delta\mathbf r \rvert}, \qquad
\text{reject if}\ \ \lvert\, \mathbf n \cdot \hat{\mathbf v}_{\text{prev}} \,\rvert > \sin\theta ,
$$

with $\mathbf r$ the ray through the pixel, $\Delta\mathbf r$ its change
for a screen step along the edge, read from the record's fitted slope, and
$\hat{\mathbf v}_{\text{prev}}$ last frame's view direction through the
reprojected position. A camera pan along a horizontal edge no longer counted.

The second refinement was for objects turning in front of a still camera.
Their contours slide just the same, while the view direction through the
pixel hardly moves: a character spinning at 1.5° per frame moves its contour
points by about 0.1° of view angle per frame. What slides the contour is the
view direction turning *relative to the surface*, so last frame's view
direction was first turned along with the surface. The surface's turn came
from the motion vectors, not from stored normals: two pixels inside the
contour give a chord whose ends are the same material points in both frames,
and the rotation taking last frame's chord to this frame's was applied to the
view direction. G-buffer normals would not have done: next to a character's
silhouette they turn by 10° to 23° per pixel, and the jitter moves each
frame's sample by up to a pixel, so normals from two frames differ by degrees
on a still character.

Both refinements worked as designed, and the test was still wrong, because
a hard threshold is all or nothing. In a camera orbit around the character
at 1280×720 and 100% screen percentage, counting the character's drawn
silhouette pixels and the share of them **detached**, with no depth step
within two pixels along their axis:

| Orbit | Test off | 2° per frame | 1° | 0.5° |
|---|---|---|---|---|
| 0.5° per frame | 4,153 px, 10.2% detached | same as off | same as off | 1,698 px, 1.6% |
| 1.5° per frame | 2,513 px, 23.7% detached | same as off | 767 px, 5.1% | |

While the camera turns slower than the threshold, nothing is rejected and
the silhouette lags: a tenth of its pixels detached at 0.5° per frame, a
quarter at 1.5°. Faster, the vertical parts of the silhouette reset every
frame and mostly vanish. The threshold only picks the turn rate at which lag
turns into absence, and sharp silhouettes, such as a box's edges, which never
slide, were rejected along with the smooth ones.

The background test has no threshold on the motion at all: a record is kept
as long as its contour is still beside it. With the records on the
foreground and this test, on the same character:

| Scenario | View-direction test, records on the background | Background test, records on the foreground |
|---|---|---|
| camera orbit, 0.5° per frame | 4,146 px, 422 detached (10.2%) | 3,264 px, 0 |
| camera orbit, 1.5° per frame | 773 px, 41 detached | 1,925 px, 0 |
| character spin, 0.5° per frame | 3,139 px, 106 detached (3.4%) | 2,742 px, 0 |
| character spin, 1.5° per frame | 1,517 px, 18 detached | 3,205 px, 0 |

No detached pixel in any scenario, and at the fast turns two to two and a
half times the silhouette. Switching the view-direction test back on under
the new layout made the silhouette vanish again at the fast orbit, down to
712 pixels on the character, so the test is gone.

## Failed Taps: Kept, Faded or Dropped

A tap that fails its depth test is not thrown away at once, and the reason
is the jitter. At a depth step, or where a sliver of another surface is less
than a pixel tall, the pixel's jittered sample lands on the other surface on
some phases of the jitter cycle. The test fails on that frame, and on the
next one too, because the stored depth is now the other surface's. If the
record were dropped, it would restart with almost no samples and the outline
would blink: when I measured what flickered in a static view of my test
level, a third of the large frame-to-frame changes of the outline were a
record joining or leaving a pixel's pool, and switching the depth test off
as an experiment, which only a static view allows, halved the flicker.

So a tap that fails its test is merged with a negligible weight, a
thousandth of its own. Next to a tap that passes it changes nothing. If no
tap passes, the weights are restored and the record is kept, marked as
failed. A failed frame counts as a frame on which the edge was not found at
the pixel, and that is what the record's presence tracks: the outline fades
over a few failed frames and the record is dropped after about a dozen of
them in a row, while a single bad phase of the cycle only dents it. The
flicker suppression post defines the presence; what matters here is that a
failed test sinks it instead of emptying the record.

A crease record kept this way keeps the depth of the surface it was recorded
on, not the pixel's current depth. Otherwise the next frame's test would
compare it against the surface the pixel shows now, pass, and the record
would be adopted by the wrong surface; in a first version that doubled the
crease pixels under a fast camera orbit, from 9,692 to 17,513. The pixel
stores one depth, so the old depth is only kept while no record of the
pixel's other axis belongs to its current surface.

Keeping failed records has a limit, the **moving gate**: on a pixel moving at
0.05 pixels per frame or faster, a failed record is dropped at once, and a
silhouette sample takes a crease record over at once, where on a still pixel
it has to wait for the crease's presence to sink. Without the gate, kept
silhouette records trailed moving contours: 6% of the drawn silhouette pixels
were detached under a pan, 11% under a camera orbit at 0.5° per frame and
19% at 1.5°. The gate is a heuristic. Parts of a spinning character near its
axis move slower than the gate, keep their failed records, and trail a
little: 0.4% to 1.2% detached pixels on the spinning character.

## The Velocity Test

### Records a Foot Leaves on the Floor

In Epic's Stack O Bot sample, the walking character left crease ghosts at
its feet: a short trail of outline on the floor behind each foot. The tests
above all let it through. The floor pixel the foot has just uncovered
fetches its history with its own velocity, which is the camera's motion,
so it lands on the pixel that showed the foot last frame and takes the foot's
crease records. The depth test passes: the foot stood on the floor, so the
depths match within 5%. No crease is found at the floor pixel, so the
record's presence sinks, but it sinks over about twelve frames and the
outline is visible for about four of them, and that is the trail. Silhouette
records have their two guards against the same slide; a crease record has
only its depth, and a crease joins two surfaces without a depth step by
definition. (A hard limit on frames without a crease removed these
footprints but blinked every crease that is found on some frames only; the
flicker suppression post tells that story.)

What differs between the foot and the floor is their **motion**: the foot's
pixel moved on its own last frame and the floor's did not. So each pixel
stores, with its records, the velocity they moved with, and a tap whose
stored velocity disagrees with the fetching pixel's is left out.

### The Object Velocity

Not the full velocity, though. Every pixel's velocity holds the camera's
motion. When the camera speeds up, slows down or turns, a mouse flick going
from 0 to 20 pixels per frame within a frame, every stored velocity differs
from this frame's by that change, static geometry included, and every tap of
the frame would fail. What is stored and compared is the **object
velocity**: the pixel's velocity minus the velocity a static point at the
same screen position and depth would have this frame, in viewport pixels per
frame.

```hlsl
float3 GetPixelVelocity(int2 PixelPos, float DeviceZ, out float2 ObjectVelocity)
{
	const float4 EncodedVelocity = SceneTexturesStruct.GBufferVelocityTexture.Load(int3(PixelPos, 0));
	const float3 StaticVelocity = ComputeStaticVelocity(ViewportPosToScreenPos(PixelPos - int2(View.ViewRectMin.xy)), DeviceZ);
	ObjectVelocity = 0.0;
	if (EncodedVelocity.x <= 0.0)
	{
		return StaticVelocity;
	}
	const float3 Velocity = DecodeVelocityFromTexture(EncodedVelocity);
	ObjectVelocity = ScreenVelocityToViewportPixels(Velocity.xy - StaticVelocity.xy);
	return Velocity;
}
```

A pixel of static geometry gets exactly 0, whatever the camera does, every
frame. A pixel of a moving object gets the screen projection of the object's
own motion, which does not depend on the camera's motion either, since both
terms are taken at the same depth. The footprint is then a tap with a
non-zero object velocity fetched by a pixel whose object velocity is 0, and a
camera acceleration changes nothing. What is left is the object's **own**
acceleration: the velocity stored last frame lags this frame's by one frame
of it. A swinging foot at 10 pixels per frame changes its speed by one or two
pixels per frame per frame, so the tolerance has a part relative to the
fetching pixel's speed: a tap is kept when

$$
\lvert\, \mathbf v_{\text{tap}} - \mathbf v_{\text{fetch}} \,\rvert
\le 0.5 + 0.5\, \lvert \mathbf v_{\text{fetch}} \rvert
$$

pixels per frame. A static pixel, the floor under the foot, allows the base
tolerance only.

```hlsl
bool IsHistoryTapOnFetchSurface(float2 TapObjectVelocity, float2 FetchObjectVelocity)
{
	const float2 Difference = TapObjectVelocity - FetchObjectVelocity;
	const float Tolerance = HistoryVelocityTolerance + HistoryVelocityRelativeTolerance * length(FetchObjectVelocity);
	return dot(Difference, Difference) <= Tolerance * Tolerance;
}
```

Which velocity a pixel stores is the **fetch's**: the one its records moved
with this frame, which is the pixel's own except on the frames when it
samples the background next to a contour and fetches with the foreground
neighbour's. That detail matters. The floor pixel the foot's contour sweeps
over fetches with the foot's velocity on that frame, receives the foot's
records, and stores the foot's velocity with them; next frame, as a plain
floor pixel, it fetches with its own, and the mismatch catches them. Had it
stored its own velocity, the foot's records would have become the floor's.

![Two rows of six square pixels, last frame above and this frame below, with a purple foot covering the left pixels of each row and its edge drawn as a thick black line, in the third pixel of the upper row and the fourth of the lower one; a label says the foot moves one pixel per frame to the right and the pixels to the right are static floor. Under the upper row's third pixel a box reads "record, stored v = 1". In the lower row the third pixel is labelled P, floor now, and the fourth Q, the foot's edge. A red dashed arrow runs straight up from P to the record box, crossed out, with the note that P is floor with object velocity 0, fetches straight up, and the tap stored with 1 is left out. A green dashed arrow runs from Q one pixel to the left up to the same record box, with a check mark and the note that Q has object velocity 1, fetches one pixel to the left, and the tap stored with 1 is kept. A caption repeats the rule: each pixel stores the object velocity its records moved with, and a tap is left out when its stored velocity differs from the fetching pixel's by more than 0.5 pixels per frame plus half that pixel's own speed](/assets/images/thin-outline/velocity-test.svg)

### Dropped, Not Faded

A tap that fails the velocity test is left out, with weight 0, unlike one
that fails the depth test. The depth test is kept soft because the jitter
fails it spuriously; the velocity test has no such source, since static
geometry is exactly 0 on both sides, and a kept record is precisely the
ghost. On a still camera the floor pixel is not moving, so the moving gate
would not drop it either, and a faded footprint would last as long as it did
before. The remaining taps keep their bilinear weights, as for taps outside
the view, so a pixel at the border of a footprint gets a lighter record
once. The test applies to both record types, because it is a test of the
tap's surface, not of its edge: a silhouette record of the foot on a floor
pixel usually fails the background test anyway, but a part of the foot
within two pixels keeps the background in reach.

I tried a second form first: move each tap forward to this frame by its own
stored velocity and keep it only if it still overlaps the pixel. Once the
camera's share of a tap's motion is taken to be the fetch's, which it has to
be for the same reason as above, that is the same comparison with the tap's
bilinear offset folded in: a tap already 0.9 pixels off on one side fails
from 0.1 pixels of extra motion, one 0.1 pixels off only from 0.9. The same
foot at the same speed was caught or missed by its sub-pixel alignment,
which alternates pixel by pixel along a trail, and in a test it cleared only
the faster of the two feet. The two forms cost the same, and the velocity
match has a threshold that means one thing: how differently from the
fetching pixel a tap's surface may have moved, in pixels per frame.

### What It Catches

In my test project, with the character spinning in place fast enough for
its feet to move 1 to 3.5 pixels per frame at 1280×720 and 100% screen
percentage, the crease record pixels around the feet went from 638 without
the test to 541 with it, and the removed ones lie along the soles' contact
line with the floor, on both feet, reproducible to one pixel between runs.
At a slower spin, with the feet under one pixel per frame, no trail forms
and nothing changes. Under a camera pan, the records on everything outside
the character are the same with and without the test, within 0.1%. In Stack
O Bot the trail behind the feet is gone.

<!-- capture: Stack O Bot, the feet's crease ghost with the velocity test off and on -->

What the test does not cover: on the first frame a foot lifts from rest its
stored velocity is still the floor's, so the floor it uncovers on that one
frame is not caught, though it is the frame after, and a lift-off uncovers
less than a pixel. An object that stops abruptly, from above the tolerance
to zero in one frame, loses its records once and rebuilds them over a few
frames. And with the test on, the records of a fast-spinning character vary
from run to run by about 5% of their pixels, because taps near the tolerance
flip between pass and fail; whether its own creases are visibly less steady
is something to judge in play, and the relative part of the tolerance is
what to raise if they are.

## The Tests at a Glance

| Test | Record type | A tap is kept when | A failing tap |
|---|---|---|---|
| sampling range | crease, on the merged record | its midline stays within $[0, 1.5]$ over the pixel's height | the record is dropped |
| depth | crease | its stored depth matches the pixel's own, moved to the previous frame, within 5% | merged at a negligible weight; if every tap fails, the record is kept as failed and its presence sinks |
| foreground in place | silhouette | the record's foreground depth matches the pixel's or its foreground-side neighbour's, within 5% | as for the depth test |
| background within reach | silhouette, on the merged record | a background lies within two pixels toward the record's background side | the record is kept as failed and its presence sinks |
| velocity | both | its stored object velocity matches the fetch's within 0.5 pixels per frame plus half the fetch's speed | left out |

On a pixel moving at 0.05 pixels per frame or faster, a record kept as
failed is dropped at once. Then the counts decay, this frame's samples are
added, and the records are stored for the next frame.

## What the Other Posts Cover

The [flicker suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %})
post defines the presence that the failed tests feed, and covers the ghosts
the depth test cannot see and the velocity test does not catch: records slid
along a surface and copies on a face widening on screen. The
[silhouettes]({% post_url 2026-10-11-thin-outline-silhouettes %}) post has
the kept ratio and the foreground depth the silhouette tests read, and the
[drawing and performance]({% post_url 2026-10-11-thin-outline-drawing-and-performance %})
post has what the history fetch costs and how its work is kept off the
pixels that have no records nearby.
