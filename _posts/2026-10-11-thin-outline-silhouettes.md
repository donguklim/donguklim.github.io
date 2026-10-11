---
layout: post
title: "Thin Outlines From the G-Buffer Under TAA: Silhouettes"
date: 2026-10-11 11:00:00 +0900
categories: [rendering, npr]
hidden: true
---

A silhouette is the outline where a surface ends and something farther away
shows behind it. Reconstructing one works like reconstructing a crease, with
the same temporal records and the same fit, but a silhouette has a front and
a back, and that changes three things: how a depth step is told apart from a
bend in the surface, which pixel records the edge and how it remembers which
side the background is on, and what the temporal upscaler has to be told so
it moves the outline with the object in front. This post covers those three
things. It is part of a [series]({% post_url 2026-10-11-thin-outline-plugin %})
and assumes the [core idea]({% post_url 2026-10-11-thin-outline-core-idea %}).

## Creases and Silhouettes

A **crease** occurs at a steep change of the normal: the edge is painted
directly onto the geometry, both sides of it belong to the same surface, and
wherever the camera goes, the same surface points form it. A **silhouette**
occurs at a depth discontinuity. It takes a foreground and a background
surface to produce one, and it is view-dependent: as the camera turns, the
silhouette slides across the surface, appearing and disappearing. The outline
is painted only on the background side, so it wraps around the foreground
object instead of eating into its shape, like an inverted hull would.

Everything in the core idea post was written for creases. From here on,
"the check" is a silhouette check unless I say otherwise, and a pixel that
finds a silhouette toward any neighbour runs no crease checks at all that
frame.

## Telling a Silhouette From a Bend

The obvious depth test, the relative depth difference between a pixel and
its neighbour, fails on surfaces seen at a grazing angle. In my test level a
floor seen at a grazing angle changes depth by 2.6% per row at 50% screen
percentage, against a silhouette threshold of 1%. Every row of the floor
read as a silhouette, and worse, the first floor row below a real
silhouette read as a second one, a row into the foreground, whose samples
also landed in the real edge's records.

So the measure works in world space. Take the target pixel C, the neighbour
N it checks toward, the neighbour O on the other side of C, and the pixel F
beyond N, all on one screen axis, and let $P$ be the world positions of
their samples and $D_C$ the linear depth of C. The measure is

$$
m = \frac{\operatorname{dist}\big(P_N,\ \operatorname{line}(P_O, P_C)\big)}{D_C} .
$$

O, C and N lie on one screen axis, so their view rays lie in one plane
through the camera, and on any plane surface the three sample points lie on
one world line, whatever the viewing angle. A grazing floor passes with a
measure of zero. Only a point that leaves the line through its neighbours,
by more than the threshold times the depth, counts as a step.

![Two panels, each showing four sample points O, C, N and F along one screen axis as dots on surfaces seen from a camera at the left. In the upper panel, labeled occlusion, O and C lie on a far surface and N and F on a near surface in front of it, and the far surface continues behind the near one. The dashed line through O and C misses N by a large gap, and the dashed line through F and N passes far in front of C, so both distances are large and the pixel is a silhouette. In the lower panel, labeled junction, O and C lie on a wall and N and F on a floor that meets the wall at a corner between C and N. The line through O and C still misses N, but the line through F and N, continued back, passes right through C, so the smaller of the two distances is near zero and the pixel is not a silhouette](/assets/images/thin-outline/silhouette-measure.svg)

That one-sided measure has a blind spot: a concave junction. Where a
camera-facing wall meets a floor seen at a grazing angle, the wall pixel C
just above the corner has the first floor pixel as its N, nearer than C and
off the wall's line by 1.3% to 2.6% of the depth in my level, so it read as
a silhouette, and the debug views showed the junction as a mix of
silhouette and crease. Nothing is hidden at a junction, though: C's surface
and N's surface *meet*. What the check has to decide is whether C's sampled
point is the background of an occluding contour, and the junction is told
apart by looking from N's side as well: continue N's surface back through F
and N, and at a junction that line passes through C, while at an occlusion
the near surface's line runs well in front of C. The measure takes the
smaller of the two:

$$
m = \frac{\min\Big(\operatorname{dist}\big(P_N,\ \operatorname{line}(P_O, P_C)\big),\ \operatorname{dist}\big(P_C,\ \operatorname{line}(P_F, P_N)\big)\Big)}{D_C} .
$$

In code, with the sky and the view border handled by returning no line:

```hlsl
// Distance of Position from the line through the samples A and B, or -1 when there is no line: A is sky, or A is B
// itself (clamped at the view rect border).
float GetDistanceFromLine(float3 Position, FThinOutlineSample A, FThinOutlineSample B)
{
	const float3 LineDirection = B.TranslatedWorldPosition - A.TranslatedWorldPosition;
	const float LineLengthSquared = dot(LineDirection, LineDirection);
	if (A.bIsSky || LineLengthSquared <= 0.0)
	{
		return -1.0;
	}
	return length(cross(Position - B.TranslatedWorldPosition, LineDirection)) * rsqrt(LineLengthSquared);
}

// Depth discontinuity between the target C and its neighbour N, with O the pixel on the other side of C and F the pixel
// beyond N, relative to C's depth.
float GetDepthEdgeMeasure(FThinOutlineSample C, FThinOutlineSample N, FThinOutlineSample O, FThinOutlineSample F)
{
	if (N.bIsSky)
	{
		return THIN_OUTLINE_FAR_DEPTH;
	}

	// A sky O makes C the foreground of a silhouette toward O, which says nothing about N.
	float Distance = GetDistanceFromLine(N.TranslatedWorldPosition, O, C);
	if (Distance < 0.0)
	{
		return 0.0;
	}

	if (bSymmetricDepthMeasure != 0)
	{
		const float DistanceFromNSide = GetDistanceFromLine(C.TranslatedWorldPosition, F, N);
		if (DistanceFromNSide >= 0.0)
		{
			Distance = min(Distance, DistanceFromNSide);
		}
	}
	return Distance / C.Depth;
}
```

A sky N is an infinite step; a sky C is the background of a silhouette
toward every non-sky neighbour; without an F, at the view border or against
the sky, only the first term counts. The strength is the measure above the
threshold, scaled and saturated, as for creases. In my test level at 100%
screen percentage the second term turned the wall-to-floor junctions into
valley creases and left the real silhouettes alone:

| Measure | Silhouette pixels | Crease pixels | Pixels whose depth is overwritten |
|---|---:|---:|---:|
| one-sided | 11,804 | 7,388 | 10,452 |
| symmetric | 9,339 | 9,102 | 8,117 |

Its side effect: a smooth surface seen at a grazing angle next to a small
depth step also counts as continuous. A pillar whose side turns toward the
wall behind it at a grazing angle and sits 2.5% of the depth in front of it
is, from depth alone, indistinguishable from a pillar standing on the wall,
and its sides became creases.

The same measure against the same threshold is the "same surface" test
everywhere else in the algorithm: whether a crease check may run across a
pair at all, whether the opposite neighbour's normal can be used, and the
search for the foreground described at the end of this post. That is also
why every pixel loads the samples two steps away along both axes.

## Which Pixel Records a Silhouette

A crease check fires for both pixels of a straddling pair, and the core idea
post builds on that: the edge pixel records through both of its checks, and
each neighbour records a copy of one half. A silhouette check is asymmetric.
Of the two pixels of a pair, one samples the foreground and one the
background, and only one of them should keep the sample.

My first design let the **background** pixel record, through the check that
faces the foreground, which is the natural reading of "the outline is drawn
on the background". It made the history expensive and fragile. A record
kept on the background has to move with the foreground, because the contour
belongs to the object in front, so every pixel searched for the foreground's
velocity up to two pixels away and ran a second history fetch for it. And
when a contour slid over a curved surface as the camera orbited, a record
that drifted into the foreground still found "its foreground" next to it and
was kept: at a 0.5° per frame orbit around the character, 10% of its
silhouette pixels were detached from any depth edge, 24% at 1.5°. A test on
the view direction's turn could cut the lag only by making the silhouette
vanish whenever the camera turned faster than its threshold.

Now the **foreground** pixel records: the check fires on C toward N when C
is nearer than N and the measure exceeds the threshold. Picture a nearly
vertical contour inside pixel B, with the foreground on the left:

![Five square pixels in a row labeled Z, A, B, C and D, with a nearly vertical black contour inside B and the foreground, shaded purple, covering Z, A and the left part of B. Two copies of the row are stacked. In the upper copy, labeled frame 1, every pixel's jitter dot sits toward the left of its square, so B's sample lands on the foreground; a red arrow from B's dot to C's dot marks B's right check firing, and B is marked as recording. In the lower copy, labeled frame 2, the dots sit toward the right, so B's sample lands on the background; a red arrow from A's dot to B's dot marks A's right check firing, and A is marked as recording. A caption notes that both records come through the right check, so the kept ratio of both is near 0: background on the right](/assets/images/thin-outline/silhouette-recording.svg)

On frames when B's sample lands on the foreground, B's right check fires and
B records; on the other frames B's sample lands on the background, A's right
check fires against it, and A records. Between them, A and B get one sample
per frame, split in proportion to how much of B lies on each side of the
contour, and the two records hold the same two halves of the edge's samples
that an edge pixel and its neighbour hold for a crease, in the same
coordinates, so the pooling of the core idea post decodes the same line from
them. Both A and B are foreground pixels, and a foreground pixel's records
move with its own velocity. That is what makes silhouette records as cheap
as crease records: one history fetch per pixel, with the pixel's own motion
vector. The only exception is the pixel at the contour on the frames when it
samples the background: its records belong to the foreground it sampled on
the other frames, so on those frames it fetches with the velocity of the
neighbour its background check fired toward.

### Which Side the Background Is On

The contour's records are indistinguishable from a crease's in their
statistics, and the half-pixel shift that folds the right check's samples
onto the left check's throws away which check they came from. For a
silhouette that direction is exactly what says where the background is, so
it is tracked separately, as one more decayed average in the record: the
**kept ratio**, the fraction of its samples recorded through the left (or
top) check,

$$
\langle k\rangle_i = (1-d)\,\langle k\rangle_{i-1} + d\,k_i, \qquad
k_i = \begin{cases}
1 & \text{sample recorded through the left (top) check} \\
0 & \text{sample recorded through the right (bottom) check,}
\end{cases}
$$

updated only on frames that record a sample, and merged with the other
statistics when records are pooled. A silhouette's foreground pixels only
ever record through the check that faces the background, so the ratio
settles near 1 when the background is on the left or top and near 0 when it
is on the right or bottom, and comparing it with one half gives the
background side. For a crease, where both checks fire, it tracks the
relative sizes of the two trapezoids and isn't used.

The record also keeps a running mean of the recording pixel's own depth,
the **foreground depth**, in half precision and in meters. The history
tests of the next post compare it with the depths under the pixel after
reprojection to decide whether the foreground is still there.

## One Record per Axis, Crease or Silhouette

A pixel draws either a silhouette or a crease, so each axis keeps a single
record and stores its type in the sign of the sample count. The type is per
axis, not per pixel: a box corner can hold a vertical crease in its
horizontal record and a horizontal silhouette in its vertical record. When
the two kinds of sample meet at one axis, the rules are asymmetric:

- A **silhouette sample takes a crease record over.** On a moving pixel at
  once; on a still pixel, a frame with a silhouette sample counts as a frame
  on which the crease was not found, so the crease's presence sinks and the
  silhouette takes over as it fades out, while a sporadic silhouette sample
  on an edge sampled mostly as a crease only dents it. The
  [flicker suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %})
  post explains the presence.
- A **crease sample is ignored by a silhouette record** unless the record
  has faded below half a sample, in which case the crease takes it over.

The second rule exists because the pixel at a contour samples the foreground
on some frames and the background on others, and the foreground may well
have creases of its own. Without the rule, every foreground frame would
reset the contour's record. The threshold has to be below one sample: a
silhouette record that just got its first sample has $n = 1$, which the
decay makes 0.96 on the next frame, so with a threshold of 1 the next
frame's crease sample would take the record over, the next silhouette sample
would take it back, and a silhouette whose foreground has creases would
never build up. At 0.5, a record with one sample keeps crease samples out
for 17 frames, and a steady one, with $n \approx 25$, for about 96 frames
without a silhouette sample. The decision, per axis, once this frame's
samples and the reprojected history of both types are in hand:

```hlsl
const bool bSilhouetteHolds = SilhouetteRecord.SampleCount > 0.0 && SilhouetteRecord.SampleCount >= SilhouetteCreaseTakeoverSampleCount;

if (bSilhouetteHolds || (bSilhouetteSample && CreaseRecord.SampleCount <= 0.0))
{
	// A silhouette check fires on the foreground pixel, toward the background.
	Record = SilhouetteRecord;
	EdgeType = EDGE_SILHOUETTE;
	FirstStrength = FirstSilhouette;
	SecondStrength = SecondSilhouette;
	SampleDepth = CenterDepth;
}
else if (bCreaseSample || CreaseRecord.SampleCount > 0.0)
{
	Record = CreaseRecord;
	EdgeType = EDGE_CREASE;
	FirstStrength = FirstCrease;
	SecondStrength = SecondCrease;
}
else
{
	// A faded silhouette record, if any, decays on.
	Record = SilhouetteRecord;
	EdgeType = EDGE_SILHOUETTE;
}
```

The history of each type is reprojected separately, each tap of the
previous frame feeding the record of its own type, so a pixel whose four
bilinear taps hold different types still gets both. In a static view of my
test level, before the two types shared a record, about 1,300 of the axis
records held both a crease and a silhouette; afterwards most of them held
the silhouette with the same total sample count, and the rest had
silhouette records that had already faded to about the threshold. Sharing
also cut the records from 76 to 44 bytes per rendering pixel.

An axis whose own record is a silhouette record draws no crease, even when
its neighbours' crease records would fit one through it. The difference is
a handful of pixels, and a pixel at a contour should not draw the
foreground's creases.

## Drawing on the Background Side

A crease's band is symmetric around the edge. A silhouette's band lies
entirely on the background side: in signed distance from the edge line it
is $[0, T]$ toward the background, with $T$ the thickness in display pixels
converted to rendering pixels, so it wraps the foreground. Which side that
is comes from the kept ratio of the pooled record.

Which records a pixel pools is the part that took a few tries. With the
records kept by the foreground pixels A and B, the band is drawn by B and
its background neighbour C, so the pool has to reach across the contour:

![Five square pixels in a row labeled Z, A, B, C and D, with the same nearly vertical black contour inside B and the foreground shaded purple on the left. A gray band of outline thickness runs along the contour on its right, the background side, filling the right part of B and the left part of C. Under the row, brackets show the pools: B pools the records of A and B, C pools the records of A and B as well, and A and D pool nothing. Small record icons sit under A and B only. A caption says the band is drawn by B and C from the records A and B hold, and that A draws nothing because the band lies entirely outside it](/assets/images/thin-outline/silhouette-band-pool.svg)

Per axis and per background side, a pixel pools the silhouette records of
itself, of the two pixels toward the foreground and of the one pixel toward
the background, as far as each record's own background lies on that side,
and it pools at all only if it or its foreground-side neighbour holds such a
record. Of the two sides, the one with more samples is drawn. So B pools A
and B, C pools A and B, A pools Z and A, finds nothing but its own record,
and its band falls entirely outside its square, and D has no record in
itself or in C and draws nothing. The band therefore can't reach further
than about one rendering pixel past the edge: only the two pixels next to
it ever draw it.

```hlsl
// The silhouette records of one background side, pooled: those of the pixel, of the two pixels toward the foreground
// side (left/top for a background along +u) and of the neighbour toward the background side, as far as their background
// is on that side. Empty unless the pixel or its neighbour toward the foreground side holds a record of that side.
FThinOutlineRecord GetSilhouetteFitRecord(FThinOutlineAxisRecords AxisRecords, bool bBackgroundAlongU)
{
	FThinOutlineRecord Pooled = GetEmptyRecord();
	const int ForegroundStep = bBackgroundAlongU ? -1 : 1;
	if (!IsSilhouetteOfSide(AxisRecords, OWN_INDEX, bBackgroundAlongU) && !IsSilhouetteOfSide(AxisRecords, OWN_INDEX + ForegroundStep, bBackgroundAlongU))
	{
		return Pooled;
	}

	UNROLL
	for (int Distance = -1; Distance <= 2; ++Distance)
	{
		const int Index = OWN_INDEX + Distance * ForegroundStep;
		if (IsSilhouetteOfSide(AxisRecords, Index, bBackgroundAlongU))
		{
			PoolRecord(Pooled, AxisRecords.Records[Index], float(OWN_INDEX - Index));
		}
	}
	return Pooled;
}
```

The two extra rules come from two failures under motion. My first rule
pooled the pixel and the two toward the foreground, and called a pixel
foreground, not allowed to draw, when its neighbour toward the background
held a record of that side. Under a steady camera pan the contour moves
about 1.6 pixels per frame, and for a while a third pixel on the background
side still holds valid records of the same edge, from the frames when it
was the foreground. The guard then stopped the pixel that contained the
edge from drawing, and one layer of the outline was missing along the
character's vertical sides. And a pixel two outside the edge that pooled
only one of the two records got a fit from half the data, whose slope alone
tilted the band into the pixel: a faint second outline row. Reaching one
pixel toward the background fixed the first, and requiring a record in the
pixel or its foreground neighbour the second.

Silhouette candidates are evaluated before crease candidates, and a pixel
with a silhouette candidate on either axis draws it; creases are only
considered when neither axis has one. That priority is also where the
minimum band area of the core idea post came from: "the band crosses the
pixel" was `area > 0`, a band that merely touched a pixel's corner left a
rounding residue of about $10^{-9}$, and such a silhouette candidate beat a
real crease of alpha 0.42 and painted nothing.

With the records on the foreground and this pooling, a test over six
scenarios at 1280×720 with TSR at 100%, counting the drawn silhouette pixels
on frame 120 and how many of them were **detached**, meaning no depth step
within two pixels along their axis:

| Scenario | Background-side records | Foreground-side records |
|---|---|---|
| static | 9,487 px, 9 detached | 10,186 px, 0 |
| camera pan, 1 cm per frame | 9,308 px, 2 | 9,298 px, 0 |
| camera orbit, 0.5° per frame | 9,237 px, 422 | 8,662 px, 0 |
| camera orbit, 1.5° per frame | 6,880 px, 251 | 7,659 px, 0 |
| character spin, 0.5° per frame | 8,974 px, 106 | 9,053 px, 0 |
| character spin, 1.5° per frame | 7,352 px, 18 | 9,516 px, 0 |

No detached silhouette pixel in any scenario, where the old layout had up to
10% on the character, and at the fast turns the silhouette stays, with two
to two and a half times the old layout's pixels on the character, where the
view-direction test had made it vanish. At the slow orbit the character
keeps about 12% fewer silhouette pixels than the old layout kept attached
ones: records that drifted are dropped instead of drawn a pixel or two off.
What drops them is the history test of the next post.

## Telling the Upscaler About the Foreground

When the outline is drawn before the temporal upscaler, a background pixel
painted with a silhouette visually belongs to the foreground object's
contour, but its depth and motion vector are the background's. The upscaler
would reproject the outline with the background's motion, and the outline
would smear or ghost whenever the foreground moved differently from what is
behind it. So the depth and velocity of every painted silhouette pixel are
overwritten with the foreground's before the upscaler runs.

Finding the foreground is a short walk. From the painted pixel, step up to
two pixels toward the foreground side of the drawn edge's axis; the near
side of the first silhouette found, by the same check as the edge detection,
is the foreground, and its device depth and its velocity are copied. Two
steps, because the pixel one outside the contour may have the pixel at the
contour sampling the background this frame, which puts the foreground two
pixels away. A pixel that samples the foreground itself finds no silhouette
and is left alone.

```hlsl
bool FindSilhouetteForeground(int2 PixelPos, bool bVertical, bool bBackgroundAlongU, out int2 ForegroundPixelPos, out float ForegroundDeviceZ)
{
	const int2 AxisStep = GetAxisStep(bVertical ? AXIS_VERTICAL : AXIS_HORIZONTAL);
	const int2 ForegroundStep = bBackgroundAlongU ? -AxisStep : AxisStep;

	FThinOutlineSample Previous = LoadDepthSample(PixelPos - ForegroundStep);
	FThinOutlineSample Current = LoadDepthSample(PixelPos);
	FThinOutlineSample Next = LoadDepthSample(PixelPos + ForegroundStep);
	ForegroundPixelPos = PixelPos;
	ForegroundDeviceZ = Current.DeviceZ;
	bool bFound = false;
	UNROLL
	for (int Distance = 1; Distance <= 2; ++Distance)
	{
		const FThinOutlineSample Beyond = LoadDepthSample(PixelPos + (Distance + 1) * ForegroundStep);
		if (!bFound && GetSilhouetteStrength(Current, Next, Previous, Beyond) > 0.0)
		{
			bFound = true;
			ForegroundPixelPos = ClampToViewRect(PixelPos + Distance * ForegroundStep);
			ForegroundDeviceZ = Next.DeviceZ;
		}
		Previous = Current;
		Current = Next;
		Next = Beyond;
	}
	return bFound;
}
```

The velocity is copied in its encoded form, which covers both kinds of
pixel: a written velocity carries the foreground's own motion, and the value
that means "derive the motion from the depth and the camera" gives, with the
overwritten depth, the foreground's camera motion. The compositing shader
can't write the depth and velocity textures it reads, so it writes the two
values to transient textures and a separate full-screen pass copies them
into the scene depth and the velocity texture the upscaler is handed, with
the depth test forced to pass. In a static frame of my test level that pass
overwrote 8,117 pixels, every one of them to the depth the compositor had
picked, and no other pixel changed. Depth of field, motion blur and the next
frame's screen-space reflections see the foreground's depth at those pixels
too.

None of this is needed when the outline is drawn after the upscaler, which
is what I do by default now: the records are still built before it, but the
line is painted at display resolution afterwards, and the upscaler never
sees the outline. The
[drawing and performance]({% post_url 2026-10-11-thin-outline-drawing-and-performance %})
post compares the two.

## What the Other Posts Cover

The [history reprojection]({% post_url 2026-10-11-thin-outline-history-reprojection %})
post has the tests that keep or drop a reprojected silhouette record: whether
its foreground depth is still under the pixel or its foreground neighbour,
whether a background is still within two pixels toward its background side,
and the view-direction test that was tried and shelved. The
[flicker suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %})
post covers the presence for silhouette records and the filter that skips
the silhouette checks of a foreground one pixel thin. Narrow background
gaps between two foreground parts, an arm in front of a torso at 50% screen
percentage, hold samples of both silhouettes in neighbouring records and
draw noise; that is a dense-edge case for the
[spatial filters]({% post_url 2026-10-11-thin-outline-spatial-filters %})
post.

Two limits of the measure remain. A foreground part one pixel thin with the
sky on its far side records no silhouette toward its near background,
because a sky O makes the measure zero. And a grazing surface next to a
small depth step, like the pillar above, is read as continuous, which is the
price of the junction term.
