---
layout: post
title: "Thin Outlines From the G-Buffer Under TAA: Silhouettes"
date: 2026-10-11 11:00:00 +0900
categories: [rendering, npr]
hidden: true
---

A silhouette is the outline where a surface ends and something farther away shows behind it. Reconstructing one works
like reconstructing a crease, with the same temporal records and the same fit. But a silhouette has a front and a
back, and that changes three things: how a depth step is told apart from a bend in the surface; which pixel records
the edge, and how it remembers which side the background is on; and what the temporal upscaler has to be told, so
that it moves the outline with the object in front. This post covers those three things. It is part of a
[series]({% post_url 2026-10-11-thin-outline %}) and assumes the
[core idea]({% post_url 2026-10-11-thin-outline-core-idea %}).

## Creases and Silhouettes

- A **crease** occurs at a steep change of the normal. The edge is painted directly onto the geometry, and both sides
  of it belong to the same surface. Wherever the camera goes, the same surface points form it.
- A **silhouette** occurs at a depth discontinuity. It takes a foreground and a background surface to produce one. It
  is also view-dependent: as the camera turns, the silhouette slides across the surface, appearing and disappearing.

The outline is painted only on a silhouette's background side. So it wraps around the foreground object, as an
inverted hull would, instead of eating into its shape.

Everything in the core idea post was written for creases. From here on, "the check" is a silhouette check unless I
say otherwise. A pixel that finds a silhouette toward any neighbour runs no crease checks at all that frame.

In short:

1. A world-space measure tells a depth step from a bend in the surface
   ([Telling a Silhouette From a Bend](#telling-a-silhouette-from-a-bend)).
2. The foreground pixels record the edge, and their records keep which side the background is on
   ([Which Pixel Records a Silhouette](#which-pixel-records-a-silhouette)).
3. Each axis keeps a single record, a crease's or a silhouette's
   ([One Record per Axis, Crease or Silhouette](#one-record-per-axis-crease-or-silhouette)).
4. The band is drawn on the background side, from records pooled across the contour
   ([Drawing on the Background Side](#drawing-on-the-background-side)).
5. When the outline is drawn before the upscaler, the painted pixels get the foreground's depth and velocity
   ([Telling the Upscaler About the Foreground](#telling-the-upscaler-about-the-foreground)).

## Telling a Silhouette From a Bend

### Why a Depth Difference Fails

The obvious depth test is the relative depth difference between a pixel and its neighbour. It fails on surfaces seen
at a grazing angle. In my test level, a floor seen at a grazing angle changes depth by 2.6% per row at 50% screen
percentage, against a silhouette threshold of 1%. As a result:

- every row of the floor read as a silhouette;
- worse, the first floor row below a real silhouette read as a second one, a row into the foreground, and its
  samples also landed in the real edge's records.

### Distance From the Neighbours' Line

My measure therefore works in world space. It uses four pixels on one screen axis:

- the target pixel C;
- the neighbour N it checks toward;
- the neighbour O on the other side of C;
- the pixel F beyond N.

Let $P$ be the world positions of their samples, and $D_C$ the linear depth of C. The measure is

$$
m = \frac{\operatorname{dist}\big(P_N,\ \operatorname{line}(P_O, P_C)\big)}{D_C} .
$$

O, C and N lie on one screen axis, so their view rays lie in one plane through the camera. On any plane surface, the
three sample points therefore lie on one world line, whatever the viewing angle. A grazing floor passes with a measure
of zero. Only a point that leaves the line through its neighbours by more than the threshold times the depth counts
as a step.

### Junctions Are Not Silhouettes

That one-sided measure has a blind spot: a concave junction. Take a camera-facing wall that meets a floor seen at a
grazing angle. The wall pixel C just above the corner has the first floor pixel as its N. That N is nearer than C,
and off the wall's line by 1.3% to 2.6% of the depth in my level. So C read as a silhouette, and the pixels along
the junction became a mix of silhouette and crease.

Nothing is hidden at a junction, though: C's surface and N's surface *meet*. What the check has to decide is whether
C's sampled point is the background of an occluding contour. The junction is told apart by looking from N's side as
well. Continue N's surface back through F and N:

- at a junction, that line passes through C;
- at an occlusion, the near surface's line runs well in front of C.

![Two panels, each showing four sample points O, C, N and F along one screen axis as dots on surfaces seen from a camera at the left. In the upper panel, labeled occlusion, O and C lie on a far surface and N and F on a near surface in front of it, and the far surface continues behind the near one. The dashed line through O and C misses N by a large gap, and the dashed line through F and N passes far in front of C, so both distances are large and the pixel is a silhouette. In the lower panel, labeled junction, O and C lie on a wall and N and F on a floor that meets the wall at a corner between C and N. The line through O and C still misses N, but the line through F and N, continued back, passes right through C, so the smaller of the two distances is near zero and the pixel is not a silhouette](/assets/images/thin-outline/silhouette-measure.svg)

The measure takes the smaller of the two distances:

$$
m = \frac{\min\Big(\operatorname{dist}\big(P_N,\ \operatorname{line}(P_O, P_C)\big),\ \operatorname{dist}\big(P_C,\ \operatorname{line}(P_F, P_N)\big)\Big)}{D_C} .
$$

### The Measure in the Shader

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

The special cases:

- A sky N is an infinite step.
- A sky C is the background of a silhouette toward every non-sky neighbour.
- Without an F, at the view border or against the sky, only the first term counts.

The strength is the measure above the threshold, scaled and saturated, as for creases.

### What the Junction Term Changes

In my test level at 100% screen percentage, the second term turned the wall-to-floor junctions into valley creases,
and left the real silhouettes alone:

| Measure | Silhouette pixels | Crease pixels | Pixels whose depth is overwritten |
|---|---:|---:|---:|
| one-sided | 11,804 | 7,388 | 10,452 |
| symmetric | 9,339 | 9,102 | 8,117 |

Its side effect: a smooth surface seen at a grazing angle next to a small depth step also counts as continuous. Take
a pillar whose side turns toward the wall behind it at a grazing angle, and which sits 2.5% of the depth in front of
the wall. From depth alone, it is indistinguishable from a pillar standing on the wall, and its sides became creases.

### The Same-Surface Test

The same measure, against the same threshold, is the "same surface" test everywhere else in the algorithm. It is used
in three places:

- the decision whether a crease check may run across a pair at all;
- the decision whether the opposite neighbour's normal can be used;
- the search for the foreground, described [at the end of this post](#finding-the-foreground).

That is also why every pixel loads the samples two steps away along both axes.

## Which Pixel Records a Silhouette

A crease check fires for both pixels of a straddling pair, and the core idea post builds on that. The edge pixel
records through both of its checks, and each neighbour records a copy of one half.

A silhouette check is asymmetric. Of the two pixels of a pair, one samples the foreground and one the background.
Only one of them should keep the sample.

### Why Not the Background Pixel

My first design let the **background** pixel record, through the check that faces the foreground. That is the natural
reading of "the outline is drawn on the background". But it made the history expensive and fragile:

- **Expensive.** A record kept on the background has to move with the foreground, because the contour belongs to the
  object in front. So every pixel searched for the foreground's velocity up to two pixels away, and ran a second
  history fetch for it.
- **Fragile.** When a contour slid over a curved surface as the camera orbited, a record that drifted into the
  foreground still found "its foreground" next to it, and was kept. With the camera orbiting the character at 0.5°
  per frame, 10% of the character's silhouette pixels were detached from any depth edge; at 1.5° per frame, 24%.

A test on the view direction's turn could cut the lag only by making the silhouette vanish whenever the camera turned
faster than the test's threshold.

### The Foreground Pixel Records

Now the **foreground** pixel records. The check fires on C toward N when C is nearer than N and the measure exceeds
the threshold. Picture a nearly vertical contour inside pixel B, with the foreground on the left:

![Five square pixels in a row labeled Z, A, B, C and D, with a nearly vertical black contour inside B and the foreground, shaded purple, covering Z, A and the left part of B. Two copies of the row are stacked. In the upper copy, labeled frame 1, every pixel's jitter dot sits toward the left of its square, so B's sample lands on the foreground; a red arrow from B's dot to C's dot marks B's right check firing, and B is marked as recording. In the lower copy, labeled frame 2, the dots sit toward the right, so B's sample lands on the background; a red arrow from A's dot to B's dot marks A's right check firing, and A is marked as recording. A caption notes that both records come through the right check, so the kept ratio of both is near 0: background on the right](/assets/images/thin-outline/silhouette-recording.svg)

Which pixel records depends on where B's sample lands:

- On frames when B's sample lands on the foreground, B's right check fires, and B records.
- On the other frames, B's sample lands on the background. A's right check fires against it, and A records.

Between them, A and B get one sample per frame, split in proportion to how much of B lies on each side of the
contour. The two records hold the same two halves of the edge's samples that an edge pixel and its neighbour hold for
a crease, in the same coordinates. So the pooling of the core idea post decodes the same line from them.

### One History Fetch per Pixel

Both A and B are foreground pixels, and a foreground pixel's records move with its own velocity. That is what makes
silhouette records as cheap as crease records: one history fetch per pixel, with the pixel's own motion vector.

The only exception is the pixel at the contour, on the frames when it samples the background. Its records belong to
the foreground it sampled on the other frames. So on those frames, it fetches with the velocity of the neighbour its
background check fired toward.

### Which Side the Background Is On

In their statistics, the contour's records are indistinguishable from a crease's. And the half-pixel shift that folds
the right check's samples onto the left check's throws away which check they came from. For a silhouette, that
direction is exactly what says where the background is. So it is tracked separately, as one more decayed average in
the record: the **kept ratio**, the fraction of its samples recorded through the left (or top) check,

$$
\langle k\rangle_i = (1-d)\,\langle k\rangle_{i-1} + d\,k_i, \qquad
k_i = \begin{cases}
1 & \text{sample recorded through the left (top) check} \\
0 & \text{sample recorded through the right (bottom) check,}
\end{cases}
$$

updated only on frames that record a sample, and merged with the other statistics when records are pooled.

A silhouette's foreground pixels only ever record through the check that faces the background. So the ratio settles:

- near 1 when the background is on the left or top;
- near 0 when it is on the right or bottom.

Comparing it with one half gives the background side. For a crease, where both checks fire, the ratio tracks the
relative sizes of the two trapezoids, and isn't used.

### The Foreground Depth

The record also keeps a running mean of the recording pixel's own depth: the **foreground depth**, in half precision
and in metres. The history tests of the next post compare it with the depths under the pixel after reprojection, to
decide whether the foreground is still there.

## One Record per Axis, Crease or Silhouette

A pixel draws either a silhouette or a crease. So each axis keeps a single record, and stores its type in the sign of
the sample count.

The type is per axis, not per pixel. A box corner can hold a vertical crease in its horizontal record and a
horizontal silhouette in its vertical record.

### When the Two Types Meet

The rules for the two kinds of sample at one axis are asymmetric:

- A **silhouette sample takes a crease record over.** On a moving pixel, it does so at once. On a still pixel, a
  frame with a silhouette sample counts as a frame on which the crease was not found. So the crease's presence sinks,
  and the silhouette takes over as the crease fades out. A sporadic silhouette sample on an edge sampled mostly as a
  crease only dents the crease's presence. The
  [flicker suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %})
  post explains the presence.
- A **crease sample is ignored by a silhouette record**, unless the record has faded below half a sample. Then the
  crease takes the record over.

#### Why Half a Sample

The second rule exists because the pixel at a contour samples the foreground on some frames and the background on
others. And the foreground may well have creases of its own. Without the rule, every foreground frame would reset the
contour's record.

The threshold has to be below one sample. A silhouette record that just got its first sample has $n = 1$, which the
decay makes 0.96 on the next frame. With a threshold of 1, the next frame's crease sample would take the record over,
and the next silhouette sample would take it back. A silhouette whose foreground has creases would never build up.

At 0.5, a record with one sample keeps crease samples out for 17 frames. A steady one, with $n \approx 25$, keeps them
out for about 96 frames without a silhouette sample.

#### The Decision in the Shader

The decision is made per axis, once this frame's samples and the reprojected history of both types are in hand:

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

### Both Types in the History

The history of each type is reprojected separately: each tap of the previous frame feeds the record of its own type.
So a pixel whose four bilinear taps hold different types still gets both.

In a static view of my test level, before the two types shared a record, about 1,300 of the axis records held both a
crease and a silhouette. Afterwards, most of them held the silhouette, with the same total sample count. The rest had
silhouette records that had already faded to about the threshold. Sharing also cut the records from 76 to 44 bytes per
rendering pixel.

### No Crease on a Silhouette Axis

An axis whose own record is a silhouette record draws no crease, even when its neighbours' crease records would fit
one through it. The difference is a handful of pixels. And a pixel at a contour should not draw the foreground's
creases.

## Drawing on the Background Side

A crease's band is symmetric around the edge. A silhouette's band lies entirely on the background side, so it wraps
the foreground. In signed distance from the edge line, it is $[0, T]$ toward the background, with $T$ the thickness in
display pixels converted to rendering pixels. The kept ratio of the pooled record says which side is the background.

### Which Records a Pixel Pools

Which records a pixel pools is the part that took a few tries. The records are kept by the foreground pixels A and B,
but the band is drawn by B and its background neighbour C. So the pool has to reach across the contour:

![Five square pixels in a row labeled Z, A, B, C and D, with the same nearly vertical black contour inside B and the foreground shaded purple on the left. A gray band of outline thickness runs along the contour on its right, the background side, filling the right part of B and the left part of C. Under the row, brackets show the pools: B and C pool the records of A and B, A pools them as well but the band lies outside it, and D pools nothing. Small record icons sit under A and B only. A caption says the band is drawn by B and C from the records A and B hold, and that A draws nothing because the band lies entirely outside it](/assets/images/thin-outline/silhouette-band-pool.svg)

A pixel doesn't know in advance on which side of it the background lies. So on each axis it tries both sides: once
assuming the background lies to the right (or below), and once to the left (or above). Every silhouette record
already says on which side its own background lies, through its kept ratio. For one assumed background side, with
the foreground on the opposite side, the pixel builds its pool by three rules:

1. **Only records that agree.** A record takes part only if its own background lies on the assumed side.
2. **Only next to a record.** If neither the pixel nor its neighbour toward the foreground holds such a record, the
   pool stays empty.
3. **Four pixels.** Otherwise, the pixel pools the agreeing records of four pixels: itself, the two pixels toward the
   foreground, and the one pixel toward the background.

The pixel builds a pool for each of the two sides and draws the side whose pool holds more samples.

In the figure, the background lies to the right of the contour, and A and B hold records of that side:

- B pools A and B: its own record and its foreground neighbour's.
- C pools A and B: its foreground neighbour B holds a record, and A is the second pixel toward the foreground.
- A pools A and B as well, since B is its one pixel toward the background. But the band falls entirely outside A's
  square, so A draws nothing.
- D pools nothing: neither D nor its foreground neighbour C holds a record.

So the band never reaches further than about one rendering pixel past the edge. Only the pixel that contains the edge
and its neighbour toward the background ever draw it.

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

### What the Two Extra Rules Are For

Rules 2 and 3, the record required next to the pixel and the reach toward the background, come from two failures
under motion.

My first rule was simpler. A pixel pooled itself and the two pixels toward the foreground, and nothing toward the
background. To keep the foreground pixels from drawing, it also had a guard. If the pixel's neighbour toward the
background held a record, the pixel was taken to be part of the foreground, and drew nothing on that side.

- **A missing layer.** Under a steady camera pan, the contour moves about 1.6 pixels per frame. For a while, a pixel
  further out on the background side still holds valid records of the same edge, from the frames when it was the
  foreground. The guard saw that record next to the pixel that contained the edge, and took this pixel for
  foreground. So the pixel that should have drawn the band drew nothing, and one layer of the outline was missing
  along the character's vertical sides. Pooling that neighbour's record, instead of using it as a guard, fixed this.
- **A faint second row.** A pixel two pixels outside the edge could pool one of the edge's two records but not the
  other. Its fit came from half the data, and its slope alone tilted the band into the pixel. That showed as a faint
  second row of outline. Requiring a record in the pixel or its foreground neighbour fixed this: a pixel two outside
  the edge has neither, so its pool stays empty.

### Silhouettes Before Creases

Silhouette candidates are evaluated before crease candidates. A pixel with a silhouette candidate on either axis
draws it; creases are only considered when neither axis has one.

That priority is also where the minimum band area of the core idea post came from. The test "the band crosses the
pixel" was once `area > 0`. A band that merely touched a pixel's corner left a rounding residue of about $10^{-9}$.
Such a silhouette candidate beat a real crease of alpha 0.42, and painted nothing.

### Measured Under Motion

I tested the records on the foreground, with this pooling, over six scenarios at 1280×720 with TSR at 100%. On frame
120, I counted the drawn silhouette pixels, and how many of them were **detached**, meaning with no depth step within
two pixels along their axis. The middle column is the old layout, with the records kept on the background side:

| Scenario | Background-side records | Foreground-side records |
|---|---|---|
| static | 9,487 px, 9 detached | 10,186 px, 0 |
| camera pan, 1 cm per frame | 9,308 px, 2 | 9,298 px, 0 |
| camera orbit, 0.5° per frame | 9,237 px, 422 | 8,662 px, 0 |
| camera orbit, 1.5° per frame | 6,880 px, 251 | 7,659 px, 0 |
| character spin, 0.5° per frame | 8,974 px, 106 | 9,053 px, 0 |
| character spin, 1.5° per frame | 7,352 px, 18 | 9,516 px, 0 |

The results:

- No silhouette pixel is detached in any scenario, where the old layout had up to 10% on the character.
- At the fast turns, the silhouette stays, with two to two and a half times the old layout's pixels on the character.
  There, the view-direction test had made it vanish.
- At the slow orbit, the character keeps about 12% fewer silhouette pixels than the old layout kept attached ones.
  Records that drifted are dropped instead of drawn a pixel or two off. What drops them is the history test of the
  next post.

## Telling the Upscaler About the Foreground

When the outline is drawn before the temporal upscaler, a background pixel painted with a silhouette visually belongs
to the foreground object's contour. But its depth and motion vector are the background's. The upscaler would
reproject the outline with the background's motion, and the outline would smear or ghost whenever the foreground moved
differently from what is behind it. So before the upscaler runs, the depth and velocity of every painted silhouette
pixel are overwritten with the foreground's.

### Finding the Foreground

Finding the foreground is a short walk:

1. From the painted pixel, step up to two pixels toward the foreground side of the drawn edge's axis.
2. The near side of the first silhouette found, by the same check as the edge detection, is the foreground.
3. Its device depth and its velocity are copied.

The walk takes up to two steps because of the pixel one outside the contour. On this frame, the pixel at the contour
may be sampling the background, which puts the foreground two pixels away. A pixel that samples the foreground itself
finds no silhouette, and is left alone.

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

### Writing the Depth and Velocity

The velocity is copied in its encoded form, which covers both kinds of pixel:

- a written velocity carries the foreground's own motion;
- the value that means "derive the motion from the depth and the camera" gives, with the overwritten depth, the
  foreground's camera motion.

The compositing shader can't write the depth and velocity textures it reads. So it writes the two values to transient
textures. A separate pass then copies them into the scene depth and the velocity texture the upscaler is handed,
with the depth test forced to pass. It draws one quad per screen tile with records nearby, not the whole screen (the
[tile classification]({% post_url 2026-10-11-thin-outline-tile-classification %}) post).

In a static frame of my test level at 100% screen percentage, that pass overwrote 8,567 pixels, every one of them to
the depth the compositor had picked, and no other pixel changed, inside or outside the 1,570 tiles of 14,400 it drew.
Depth of field, motion blur and the next frame's screen-space reflections see the foreground's depth at those pixels
too.

### Drawing After the Upscaler

None of this is needed when the outline is drawn after the upscaler, the other drawing mode; drawing before the
upscaler, with this overwrite, is the default. In that other mode, the records are still built before the upscaler,
but the line is painted at display resolution afterwards, and the upscaler never sees the outline. The
[drawing after the upscaler]({% post_url 2026-10-11-thin-outline-drawing-after-upscaler %})
post compares the two.

## What the Other Posts Cover

- The [history reprojection]({% post_url 2026-10-11-thin-outline-history-reprojection %}) post has the tests that
  keep or drop a reprojected silhouette record. They check whether its foreground depth is still under the pixel or
  its foreground neighbour, and whether a background is still within two pixels toward its background side. The post
  also has the view-direction test that was tried and shelved.
- The [flicker suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %}) post covers the presence for
  silhouette records, and the filter that skips the silhouette checks of a foreground one pixel thin.
- Narrow background gaps between two foreground parts, such as an arm in front of a torso at 50% screen percentage,
  hold samples of both silhouettes in neighbouring records, and draw noise. That is a case for the dense edge
  suppression of the [flicker suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %}) post.

Two limits of the measure remain:

- A foreground part one pixel thin, with the sky on its far side, records no silhouette toward its near background,
  because a sky O makes the measure zero.
- A grazing surface next to a small depth step, like the [pillar above](#what-the-junction-term-changes), is read as
  continuous. That is the price of the junction term.
