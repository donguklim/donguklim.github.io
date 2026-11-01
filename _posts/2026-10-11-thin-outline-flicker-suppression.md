---
layout: post
title: "Thin Outlines From the G-Buffer Under TAA: Flicker Suppression"
date: 2026-10-11 14:00:00 +0900
categories: [rendering, npr]
hidden: true
---

Drawn after the upscaler, the outline no longer fades or breaks into dashes. But on some surfaces, at some distances,
it flickers: a line is there on one frame and gone on the next. This post covers what flickers and why, and how I
measured it over the jitter cycle. It covers the two suppressions I tried first, which remove tangles of dense edges
and isolated specks but did not stop the flicker. Then it covers what did: the presence of a record, a filter for
normal spikes at sampling time, and a filter for foregrounds one pixel thin. It is part of a
[series]({% post_url 2026-10-11-thin-outline %}) and assumes the
[core idea]({% post_url 2026-10-11-thin-outline-core-idea %}),
[history reprojection]({% post_url 2026-10-11-thin-outline-history-reprojection %}) and
[spatial filters]({% post_url 2026-10-11-thin-outline-spatial-filters %}) posts.

## What Flickers

### The Symptom

Drawn after the upscaler, parts of the outline blink. A line, or a piece of one, is drawn on some frames and missing
on others, and the pattern repeats with the **jitter cycle**. TSR moves the sample position inside the pixel through a
fixed sequence of $L$ positions, then starts over: $L = 11$ at 100% screen percentage, 17 at 73%, 37 at 50%. At 60
frames per second, a 17-frame cycle gives about three and a half blinks per second. That is what makes it look like
flicker rather than noise.

Drawn before the upscaler, the same blinking happens. But the upscaler blends every pixel with its own history, so a
line that is missing on 5 of 17 frames becomes a steady line at about 70% of its strength. Drawing after the upscaler
takes that blending away, along with the fading it caused. The records are then the only memory the outline has, and
every frame's decision shows as it is.

The video below is Epic's City Sample with the outline drawn after the upscaler and every step of this post turned
off: the dense and isolated edge suppression, the presence and the spike filter. It is the reference for the videos
further down, which turn the steps on one at a time.

<iframe style="width: 100%; aspect-ratio: 16 / 9; border: 0;" src="https://www.youtube-nocookie.com/embed/YHStMWZe82s" title="City Sample, every flicker suppression off" allow="accelerometer; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

*The City Sample, outline drawn after the upscaler, every flicker suppression off.
[Watch on YouTube](https://youtu.be/YHStMWZe82s).*

### How I Measured It

The flicker repeats with the jitter cycle, so it can be measured in a static view.

- **The spot.** I picked a spot in my test level where the flicker was easy to see: a few ramps, walls and a pillar
  seen through a purple fog volume. The fog only makes the black lines stand out; the flicker is the same without it.
- **The render.** I rendered it offscreen at 1280×720 with 73% screen percentage, the percentage the editor picks on
  my display, so $L = 17$.
- **The count.** I dumped the outline alpha of 36 consecutive frames, about two jitter cycles. Inside a fixed region
  without the animated character, I counted the display pixels whose alpha swings by more than 0.3 over those
  frames, and those whose alpha swings by more than 0.5.

Two runs of the same build gave identical counts, so any difference between two settings was real. And a Python copy
of the drawing code reproduced the GPU's alpha exactly, so every jump could be traced back to the records behind it,
pixel by pixel.

![A flicker map of the test region at twice its size: the outline of walls, ramps and a tall pillar drawn as thin white and grey lines on black, with a dark grey box in the lower middle where the animated character is left out. Most lines are steady white; short stretches are coloured yellow to red where the alpha swings by more than 0.1 over 36 frames, most of them on the slanted ramp edges in the lower right, at the junctions where ramps meet walls, at a vertical line beside the pillar, at the pillar's top corner and along a line at the far right](/assets/images/thin-outline/flicker-map.png)

*The measured region at the start, twice its size. White and grey: a steady outline. Yellow to red: display pixels
whose alpha swings by more than 0.1 over 36 frames, red up to 1. The dark box is the character, left out.*

At the start, the count was **256** pixels swinging by more than 0.3 and **107** by more than 0.5.

### What It Is Not

- **Not the edge strength.** 94% of the drawn pixels had a mean strength of exactly 1. The strength saturates on
  almost every check that fires, and none of the large jumps I classified came from a change of strength.
- **Not the small shimmer.** Every line's alpha also wobbles by 0.05 to 0.1 from frame to frame, because each new
  sample moves the fitted line a little. Halving the decay rate removes most of that wobble (pixels wobbling by more
  than 0.05: 4,464 against 598). But it is a different thing from the blinking, and it does not look like flicker.

### Decisions That Flip

For each rendering pixel, the drawing makes several all-or-nothing decisions every frame:

- is there an edge with a valid fit;
- which axis is drawn;
- crease or silhouette.

A decision is stable when the records behind it are strong, with many samples that agree. The blinking is where they
stay weak: the samples of one edge are split up, or the records keep being reset. The decision then flips with the
jitter phase.

I sorted the 515 large jumps of the first measurement, alpha changes of more than 0.25 between two frames, by what
changed in the records behind them:

| Cause | Share |
|---|---:|
| A record joined or left the pixel's pool: reset by a history test, taken over by the other type, or new | 30% |
| The drawn type switched between crease and silhouette | 22% |
| The drawn axis switched | 17% |
| The band missed the pixel, or the pixel's own record was of the other type | 13% |
| The fit appeared or vanished at the three-sample minimum | 12% |
| The line moved: new samples, the spatial filter | 5% |
| The strength changed | 0% |

### Five Causes

Behind those shares are five causes.

1. **Records reset every jitter cycle.** At a depth step, or where a sliver of another surface is less than a pixel
   tall, the jittered sample lands on the other surface on some phases. The history depth test then fails, and the
   record restarts with almost no samples. Switching the crease depth test off as an experiment, which only a static
   view allows, halved the count. The
   [history reprojection]({% post_url 2026-10-11-thin-outline-history-reprojection %}) post has the rule this led
   to: a silhouette record is kept through a frame on which its contour is missed, while a record whose surface's
   depth is gone from under it is dropped.
2. **One edge sampled as two types.** A depth step next to a surface seen at a grazing angle is close to the
   silhouette threshold. So the checks call it a silhouette on some phases and a crease on others. Each axis keeps
   one record, so each type kept replacing the other. The records of one such line held 1 to 5 samples instead of
   about 25.
3. **Geometry smaller than a pixel.** Slivers of a third surface between two others, and tips where the foreground
   covers the pixel on some phases only.
4. **The three-sample minimum.** At 73%, a fit needs a pooled sample count of about 4.4 to hold three distinct
   samples. A pool that hovers around it switches the fit on and off. I simulated fading the outline in over the
   distinct-sample count instead, and it was worse: 98 flickering rendering pixels with the fade, against 47 with the
   hard minimum. These records get their samples on some phases only, so their count rises and falls within every cycle.
   The fade turns that into a pulsing line, while the hard minimum only flips when the count crosses it.
5. **Two edges in one pixel.** At a corner, both axes have a converged edge with nearly the same slope error. The rule
   then prefers the smaller slope, and both slopes are noise around zero: a coin toss every frame. And a single stray
   silhouette sample on the other axis gives a valid silhouette candidate whose band barely touches the pixel.
   Silhouettes are chosen first, so it hides a converged crease for that frame.

### After the First Fixes

The first cause was the largest. Over the following days, two changes brought the count from 256 and 107 down to 103
and 34:

- keeping records through failed tests, which today means through a missed contour only;
- the takeover rules between the two types, which the
  [silhouettes]({% post_url 2026-10-11-thin-outline-silhouettes %}) post describes.

What was left showed most in Epic's City Sample, on window frames, facade tiles and grooved surfaces at a distance.
Against it, I first tried two suppressions on the drawn edges, and then went back to the records. The rest of this
post follows that order:

1. [Dense Edge Suppression](#dense-edge-suppression) and [Isolated Edge Suppression](#isolated-edge-suppression): the
   two suppressions on the drawn edges.
2. [What the Presence Is For](#what-the-presence-is-for) and [The Presence](#the-presence): the fraction of recent
   frames on which a record's edge was found, and how the drawing uses it.
3. [The Normal-Spike Filter](#the-normal-spike-filter) and [The Thin Foreground](#the-thin-foreground): two filters
   at sampling time.
4. [What Is Left](#what-is-left), and a [Summary](#summary) of every step.

## Dense Edge Suppression

### The Problem

When many edges crowd into a small area of the screen, the outline flickers. I saw it on a rolling shutter (dense
silhouettes) and on the tiles of a building (dense creases), each from a certain distance.

The cause: once edges sit only a pixel or two apart along a pixel's axis, the jitter decides which of them each
pixel's checks pick up on a given frame. The records of the pixels between them mix the samples of several edges, and
the line fitted from a mixed record wanders from frame to frame. Drawn before the upscaler, the outline never settles,
and the upscaler blurs it. Drawn after the upscaler, the wandering is visible as it is.

My first rule, for a pixel with a horizontal-axis edge, looked at the two pixels to its left and the two to its
right. It suppressed the edge when:

- both distance-1 neighbours also contained an edge of the same axis, since three in a row cannot be one thin edge;
- or either distance-2 neighbour did, since two edges whose two-pixel-wide detection bands overlap mix their samples
  in the records between them.

One distance-1 neighbour alone was not dense: a single edge near the shared boundary shows up in both pixels. The rule
in the plugin keeps the reach and changes how the edges are counted. It starts from one fact about the records.

### Where One Edge's Records Are

Take a vertical edge at $E = p + e$ inside pixel $p$, with $e \in [0, 1)$, and the jitter $j \in [0, 1)$ along the
axis. Three pixels hold records of this one edge:

- pixel $p$ gets samples from both of its checks on every frame;
- $p - 1$ gets right-check samples on the frames with $j > e$;
- $p + 1$ gets left-check samples on the frames with $j < e$.

A record stores the trapezoid midline $u = \tfrac12 + E/2$ in its own pixel's frame. The fitted line passes through
the mean point, so the record's mean $\mu_u$ decodes to the edge's position $2\mu_u - 1$ at the record's mean $v$
exactly. No slope is needed to ask *where* a record's edge is:

- $p$'s record decodes to $e$, inside $p$;
- $p - 1$'s decodes to $1 + e$ and $p + 1$'s to $e - 1$, both **outside** those pixels, on $p$'s side.

![Two rows of five square pixels along the horizontal axis. In the upper row, labelled one edge, a thick black vertical edge stands inside the middle pixel p at 0.7 of its width, and record boxes under p minus 1, p and p plus 1 read right-check samples, both checks and left-check samples; dashed red arrows from each box point at the edge's foot, and the labels say the records decode to 1.7, outside, to 0.7, inside, and to minus 0.3, outside, so only p contains the edge. In the lower row, labelled two edges two pixels apart, edges E1 and E2 stand at 0.7 inside pixels p and p plus 2, and record boxes under the five pixels read E1 right, E1 both, E1 left plus E2 right, E2 both and E2 left; the verdicts under them read drawn, drawn because it contains E1, mixed and suppressed, drawn because it contains E2, and nothing to draw, and the caption explains that p plus 1 sees an edge inside the pixel on each side whose fit crosses elsewhere than its own. A footer notes that a record decodes to the edge position two mu u minus one at its mean v, with no slope needed](/assets/images/thin-outline/dense-edge-records.svg)

So a record in a pixel does not mean an edge in that pixel. A pixel next to an edge holds that edge's record, and the
record decodes to a position outside the pixel. "The pixel contains an edge" is a test on the record's mean:

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

Along the drawn edge's axis, a side (left or right; top or bottom for the vertical axis) holds **another edge** when
any of these holds, tested cheapest first:

1. **Distance 2: the pixel contains an edge.** Its own record, of either type, decodes to a position inside it. It
   cannot be this pixel's edge: the selected axis has $\lvert b \rvert \le 1$, so one edge cannot lie inside two
   pixels two apart.
2. **Distance 1: a record of another kind.** The neighbour's own record is of the other type, or a silhouette record
   of the other background side. It cannot be this pixel's edge.
3. **Distance 1: a fit of the same kind that puts the edge elsewhere.** This test runs only when the neighbour's own
   record decodes to a position inside the neighbour. Otherwise that record is this pixel's edge seen from that side,
   or the edge two steps away, which the first test covers. The test compares the centre crossing of the neighbour's
   **pooled fit** of the drawn kind, what that pixel would draw, with this pixel's. Both crossings are taken in this
   pixel's coordinates (the neighbour's plus or minus 1).

In the third test, the two count as **one edge** when

$$
\lvert u_q \pm 1 - u_p \rvert \le t \sqrt{\mathrm{SE}(u_p)^2 + \mathrm{SE}(u_q)^2},
\qquad t = 2 ,
$$

with the standard errors those of the centre crossings, the quantity the
[spatial filter]({% post_url 2026-10-11-thin-outline-spatial-filters %}) weighs by. Within the tolerance, the two
pixels have reconstructed the same edge with some error. Outside it, they hold two edges.

```hlsl
const float NeighbourCrossing = NeighbourFit.Intercept + 0.5 * NeighbourFit.Slope + float(Side);
const float DifferenceVariance = OwnCrossingVariance + Pow2(max(NeighbourFit.CrossingStandardError, THIN_OUTLINE_MIN_STANDARD_ERROR));
return Pow2(NeighbourCrossing - OwnCrossing) > Pow2(DenseEdgeTolerance) * DifferenceVariance;
```

The third test replaces my first rule's "both distance-1 neighbours". A neighbour that draws the same edge is not
evidence of density, however many of them there are.

The first test's containment condition came from a wrong first version, which counted *any* record two steps away. A
pixel drawing the band near its boundary then counted its own edge's record, two steps away in the next pixel but
one, as another edge. With the either-side trigger below, that version removed 56% of the outline alpha of my test
scene; with the containment test, 17%.

### The Trigger

The edge is suppressed when **both** sides hold another edge. That is stricter than my first rule, which suppressed
on a distance-2 edge on one side alone. The either-side trigger is kept as an option. The table shows what the two
triggers do to the simplest cases: parallel vertical edges, with the records as the checks lay them down.

| Case | Records | Both sides | Either side |
|---|---|---|---|
| An isolated edge in $p$, its band also crossing $p - 1$ | $p - 1$ (outside), $p$ (inside), $p + 1$ (outside) | $p$ and $p - 1$ drawn: $p - 1$ sees $p$'s record inside $p$, same kind, same crossing; $p + 1$'s record decodes outside $p + 1$ | the same |
| Two edges 1 px apart, in $p$ and $p + 1$ | $p - 1$ to $p + 2$, with $p$ and $p + 1$ mixed | nothing suppressed: $p + 2$'s record decodes outside $p + 2$, and the two mixed pools agree within their inflated errors | nothing |
| Two edges 2 px apart, in $p$ and $p + 2$ | $p - 1$ to $p + 3$, with $p + 1$ mixed | $p + 1$ suppressed, $p$ and $p + 2$ kept | $p$, $p + 1$ and $p + 2$ suppressed |
| Three or more edges 1 to 2 px apart | all | the inner pixels suppressed | all suppressed |

The both-sides trigger keeps an isolated pair of lines, and hides the mixed pixel between them when there is one. It
removes lines only where three or more edges crowd. Two cases slip through it:

- **Geometry one pixel wide.** A foreground strip or a thin spike whose two edges are a single pixel apart is an
  isolated pair. That is why its samples are filtered at sampling time instead, further down in this post.
- **A narrow background gap.** The gap between two foreground parts, from the
  [silhouettes]({% post_url 2026-10-11-thin-outline-silhouettes %}) post, is the same case for silhouettes. The gap
  pixel's neighbour on the far side holds a silhouette record of the other background side, which the second test
  counts. But the near side holds the gap's own contour. So the both-sides trigger leaves the gap alone, and only the
  either-side trigger removes it, with much else.

### What It Removes and Costs

In my test scene (static, 1280×720, 73% screen percentage, drawn after the upscaler), the outline draws 19,334
display pixels. What the two triggers remove:

| Trigger | Outline alpha removed | Display pixels touched |
|---|---:|---:|
| Both sides | 7.1% | 2,241 |
| Either side | 16.7% | 5,030 |

Under the both-sides trigger, the removed pixels are:

- the dense creases on the character's torso and hands;
- a few junctions where ramp edges meet;
- the inner of two parallel lines along a pillar's edge.

Every silhouette stays.

What the both-sides trigger costs, by drawing mode:

| Drawing mode | Off | Both-sides trigger |
|---|---|---|
| after the upscaler, 73% | 0.139 ms | 0.147 ms (+6%) |
| before the upscaler, 100% | 0.141 ms | 0.147 ms (+4%) |

The tests run cheapest first, and the second side is tested only when the first leaves the rule open. For an isolated
edge, no pooled fit is computed at all. A first version computed both sides' pooled fits for every drawn pixel, and
cost +14% at 73%. The cost was the fits and the record copies, not a load round trip.

### Not a Flicker Fix

The suppression is not a fix for the flicker. In Epic's City Sample, the window frames and facade tiles at a distance
flicker, and it did not help there. A mixed record's wandering is only one of the things that flicker, and not the
main one.

I kept it anyway. Where edges are dense, it stops the spot from being darkened by a tangle of lines, which a user may
or may not want, and it is a switch.

The video below has the dense edge suppression on, together with the presence and the spike filter, which come later
in this post. The same view with those two alone is the spike filter's video further down.

<iframe style="width: 100%; aspect-ratio: 16 / 9; border: 0;" src="https://www.youtube-nocookie.com/embed/nFosv3O95fo" title="City Sample, presence, spike filter and dense edge suppression on" allow="accelerometer; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

*The City Sample with the presence, the spike filter and the dense edge suppression on.
[Watch on YouTube](https://youtu.be/nFosv3O95fo).*

## Isolated Edge Suppression

The opposite case is an edge that is too *small*. An edge of one or two pixels is a speck: a record that comes and
goes with the jitter rather than a line, and it flickers.

The rule: a drawn edge must run over at least three pixels across its axis, this one included. It must continue into:

- both neighbours across the axis, or
- one neighbour and the pixel beyond it, which is an edge end or a corner.

"Continues" means that the pooled record of the drawn kind at that pixel, pooled as that pixel pools it, can be
fitted, with the fit's minimum of three distinct samples and some spread across the axis. These are the two
neighbour fits of the [spatial filter]({% post_url 2026-10-11-thin-outline-spatial-filters %}). So the test starts
from them, and loads one more pooled record only when exactly one neighbour continues:

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

**What it removes and costs.** In the test scene at 73% after the upscaler, it removes 0.6% of the outline alpha, 355
display pixels. It costs 0.002 ms on top of the filter (0.0005 ms before the upscaler at 100%).

**Why it stayed.** I first built it as a flicker remedy for the City Sample. Removing one- and two-pixel edges did
nothing for the flicker there, so I took it out. It came back once the presence and the spike filter below were in.
Some surfaces with grooves still flickered at certain distances. The presence ramp could not be tuned for them without
making other creases at the same distance dashed, and the specks were the handle that was left. Sharing its loads with
the filter made it nearly free, which is the other reason it stayed.

With every step of this post on, the isolated edge suppression included:

<iframe style="width: 100%; aspect-ratio: 16 / 9; border: 0;" src="https://www.youtube-nocookie.com/embed/b3blImFzETA" title="City Sample, every flicker suppression on" allow="accelerometer; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

*The City Sample with every flicker suppression on: the presence, the spike filter, the dense and the isolated edge
suppression. [Watch on YouTube](https://youtu.be/b3blImFzETA).*

## What the Presence Is For

### Records That Pass Every Test

A crease joins two surfaces without a depth step, by definition. So the depth test of the history only notices a
crease record that lands on a different surface. A record that slides along one connected surface passes it:

- the crease records of a foot, left on the floor it stood on;
- a junction's crease, carried onto the curved wall beside it as the camera uncovers the wall;
- the copies a face gets from the crease beside it while it turns toward the camera and widens on screen, since the
  history only shifts records.

The [history reprojection]({% post_url 2026-10-11-thin-outline-history-reprojection %}) post describes all three.
Such a record has to go, because no crease is found where it is any more.

### Found on Some Frames Only

"Found" has to be judged at the record's **own pixel**, along its axis, and not on every frame.

A ledge narrower than a pixel, where two parallel walls meet, is a crease only on the frames when a jittered sample
lands on it. On the others, every normal around it is the same, and no test at any threshold can see it. And the
pixel beside a crease gets samples only on the frames when its own pair of pixels straddles the crease. Those are
exactly the frames on which its own check finds it.

So a record of either kind is found on some frames of every jitter cycle and missed on the others. A rule that drops
a record after a few misses in a row blinks it. A fade that follows the length of the current run of misses pulses
with the cycle. What tells a real edge from a ghost is the fraction of recent frames on which it is found, not the
latest run.

### Pops on Distant Grooves

The other thing that passes every test is a pop. In the City Sample, the flicker that was left sat on grooved
surfaces seen from far away, and it looked like dots.

The normal of such a surface is smooth, except for a few pixels per frame that **pop**: at the grooves, in different
places each frame. The cause is a groove wall thinner than a pixel, hit by the jittered sample on some frames. To the
checks, a pop is a crease at the strength of a sharp edge, since the groove wall's angle is steep whatever the
distance.

After two or three pops at one pixel, the record has a valid fit and draws at full strength. A record's strength is
the mean strength of its samples, and knows nothing of the frames in between. As long as nothing removes it, the dot
stays for seconds, and the pops add up to a sprinkling of dots across the surface.

A pop is found on few frames, which is again what the fraction of recent frames measures.

## The Presence

### Definition

Each record, crease or silhouette, keeps a **presence** $p \in [0, 1]$: the fraction of recent frames on which its
edge was found at its pixel. The presence is updated after the history is reprojected and before this frame's samples
are added, with $f = 1$ on a frame on which the edge is found and $f = 0$ otherwise:

$$
p \leftarrow p + \frac{f - p}{N}, \qquad N = 4 ,
$$

an exponential average over the last four frames or so. A record created this frame starts at $p = 1/N$, found once.
A record whose presence falls below 0.05 is emptied.

```hlsl
void UpdatePresence(inout FThinOutlineRecord Record, bool bFound)
{
	if (Record.SampleCount <= 0.0)
	{
		return;
	}
	Record.Presence += ((bFound ? 1.0 : 0.0) - Record.Presence) * PresenceRate;
	if (Record.Presence < PresenceDropLevel)
	{
		Record = GetEmptyRecord();
	}
}
```

The presence is stored in the fourth channel of a record's first texel. It is merged like the record's other values,
as a mean weighted by sample count: across the history taps, and when a pixel pools its record with its neighbours'.

### What Counts as Found

**Found** means two things. First, the edge is there:

- **For a crease record,** its pixel has a crease measure above half the detection threshold, toward either neighbour
  along the record's axis. This test also runs on frames when the pixel has a silhouette and records no crease.
- **For a silhouette record,** its pixel has a depth edge measure above half the silhouette threshold, toward either
  neighbour along the axis, whichever side is nearer. So the contour pixel finds its silhouette both on the frames
  when its sample is on the foreground and on those when it is on the background.

Second, the background test passed this frame. A silhouette record whose contour was not seen counts as not found.
That is how the kept records of the history post fade. A tap that fails a depth test, on the other hand, is left out
before the presence is asked.

A crease record also loses the frame to a silhouette sample. That is how a silhouette takes a crease's axis over on a
still pixel. A stray silhouette sample once per jitter cycle only dents the crease. A contour that moves in samples a
silhouette on every frame, and takes over as the crease's presence sinks.

### The Draw Ramp

The record's strength stays the mean strength of its samples. What changes is the drawing. The candidate's strength
is multiplied by a ramp on the pooled record's presence: 0 up to $p = 0.3$, rising linearly to 1 at $p = 1$:

```hlsl
Candidate.Strength *= saturate((Record.Presence - PresenceDrawMin) / max(PresenceDrawMax - PresenceDrawMin, 1.0e-4));
```

Keeping the two apart keeps the strength meaningful and the ramp tunable. Adding a zero strength to the record's mean
on every frame without the edge would merge them into one number. The decay would then average that number over tens
of frames rather than four.

What the ramp gives, for four kinds of edge:

![Four small charts over frames 0 to 15, each with the presence p as a purple line with dots, green bars for the drawn strength, and a dashed line at p = 0.3 where the ramp starts. A new edge found on every frame: p rises from 0.25 through 0.44, 0.58 and 0.68 toward 1, and the bars grow from 0 through 0.2, 0.4 and 0.55 to 0.89 at frame 8. A junction crease found on every other frame: p alternates between 0.57 and 0.43 and the bars between 0.39 and 0.18. A groove pop found on one frame in four: p jumps up on each pop to 0.33, 0.35 and 0.36 and sinks in between to about 0.15, and the bars are 0.04, 0.08 and 0.09 on the pop frames and 0 otherwise. A ghost found until frame 3 and then never: p is 1 until frame 3, then 0.75, 0.56, 0.42, 0.32 and on down to 0.06 at frame 13, the bars 1 until frame 3, then 0.64, 0.38, 0.17, 0.02 and 0, and the record is gone from frame 14. A caption repeats the update rule, the start value, the drop level and the ramp](/assets/images/thin-outline/presence-cases.svg)

| Edge | Presence | Drawn strength |
|---|---|---|
| A new edge, found on every frame | 0.25, 0.44, 0.58, 0.68, … → 1 | 0, 0.20, 0.40, 0.55, …: 0.9 after about nine frames |
| A junction crease, found on every other frame | 0.57 and 0.43 in turn | 0.39 and 0.18 in turn, instead of drawn and missing |
| A groove pop, on one frame in four | up to 0.36 right after a pop | at most 0.09, on the pop frames |
| A groove pop, on one frame in six | up to 0.30 | 0.01 at most |
| A ghost, found on every frame, then never | 1, 0.75, 0.56, 0.42, 0.32, … | 1, 0.64, 0.38, 0.17, 0.02: faint after three frames, emptied on the eleventh |

A pop's record also needs three distinct samples before it can be drawn at all. So a groove popping on one frame in
four draws at most a trace.

#### Where the Ramp Ends

I first had the ramp end at $p = 0.6$, so that a crease found on most frames drew at full strength and a new one faded
in over three frames. After trying it in the City Sample, I moved the end to 1. Now:

- only an edge found on every recent frame draws at full strength;
- an edge found on half of them draws at about 0.3;
- a new edge fades in over about nine frames.

The junction crease shows the cost: it still alternates, but between two faint strengths, 0.18 and 0.39, rather than
between drawn and missing.

### Ghosts, Failed Records, and the Cost

A ghost is drawn for three or four frames, fading, and is emptied on the eleventh frame without its edge. A silhouette
record whose contour the checks missed this frame is treated the same way. It is dropped after about a dozen such
frames in a row, and recovers from a single bad phase per cycle.

Some records skip the fade:

- **On moving pixels,** a silhouette record whose contour was missed is still dropped at once, by the moving gate of
  the history reprojection post.
- **A record whose depth test fails** is left out at any speed, before the presence is asked.
- **The footprints of a walking character** are left out too. They are the one ghost for which three or four faint
  frames were still too many, and the velocity test of that post removes them.

**The cost.** The presence costs about 0.002 ms at 1280×720 and 73%, or less than I could measure. The update is a
multiply-add and a compare per record, and the ramp is one multiply in the drawing.

**The effect on my test level.** On my static test level, the effect is mostly dimming. Against the outline drawn
with the presence ignored, 2,315 display pixels differ by more than a tenth of full strength, and 2,300 of them are
fainter. That is the ramp to 1 doing what it says to every edge found on fewer than all recent frames. Of the crease
records, 36% are found on nine frames in ten or more, and 2.9% on fewer than one in ten. Those rare ones are the sides
of creases that straddle them on some phases only, and records on their way out.

**The effect on the flicker.** I judged this on screen, in the City Sample, where the grooves are. My test level has
none, and the count of the measured region says nothing about dots on a distant facade.

### Why a Rarely Found Edge Is Not Drawn

Turning a rarely found edge down to nothing, rather than drawing it at a lower strength or holding it for a few
frames, is a choice about what the outline should show. The City Sample's normal buffer is my reason for it. The
video below shows the normal buffer as colours while the camera moves over the city:

<iframe style="width: 100%; aspect-ratio: 16 / 9; border: 0;" src="https://www.youtube-nocookie.com/embed/c78VzmgZvB0" title="City Sample, normal buffer visualization" allow="accelerometer; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

*The City Sample's normal buffer. [Watch on YouTube](https://youtu.be/c78VzmgZvB0).*

The spots where the outline's creases flicker are the spots where the normal buffer has flickering dots. Apart from
those dots, the normals there are nearly constant.

The ground with grooves is an example:

- **Seen from near,** its grooves are visible as creases in the shaded image, even with the plugin off. Its normals
  have no flickering dots: each groove is wider than a pixel, and every frame samples it.
- **As the camera moves away,** the grooves shrink below a pixel and fade out of the shaded image. All that is left
  of them in the normal buffer is the dots.

An outline drawn from those dots would draw a crease that the image itself no longer shows. So an edge found only on
a few recent frames should not be drawn at all. That is what the ramp's zero below a presence of 0.3 does.

The presence alone, against the reference with everything off at the top of this post:

<iframe style="width: 100%; aspect-ratio: 16 / 9; border: 0;" src="https://www.youtube-nocookie.com/embed/gyRO8wb87-k" title="City Sample, presence on" allow="accelerometer; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

*The City Sample with only the presence on. [Watch on YouTube](https://youtu.be/gyRO8wb87-k).*

## The Normal-Spike Filter

The presence keeps a pop from drawing. But each pop still gives three records a sample at a sharp edge's strength:
the spike's and its two neighbours'. A groove that pops on one frame in three draws faintly. The cleaner fix is not
to sample a pop at all. That is the idea of filtering thin geometry at sampling time from my
[first post]({% post_url 2026-09-06-taa-compatible-toon-outline-gbuffer-contour %}), applied to the normal.

### Which Checks Are Skipped

A pop is a **spike** in the normal field: a pixel whose normal differs from both of its neighbours along an axis,
while those two agree. Take a spike $S$ between flat pixels $L$ and $R$. Four checks fire:

- both of $S$'s checks, since the normal turns and turns back;
- $L$'s check toward $S$ and $R$'s toward $S$, since there is no turn on their other side to carry on.

So a check is skipped when:

- **the pixel is the spike:** it has a crease toward both of its neighbours along the axis, and those two are one
  surface; then both of its checks are skipped;
- **the neighbour is the spike:** the pixel has a crease toward that neighbour, and the pixel and the one beyond the
  neighbour are one surface; then the check toward the neighbour is skipped.

A skipped check records nothing. Its crease measure is cleared too, so the presence does not count it as found.

The second case needs the normal of the pixel two steps away, which the sampling otherwise does not load. It is
loaded only where the pixel has a crease toward that neighbour and is not a spike itself: a few percent of the
pixels.

![Three side views with the camera on the left and three dashed horizontal rays, one per pixel row v minus 1, v and v plus 1, ending in a dot on the first surface they hit, with a blue arrow for the surface normal there. In the first, a groove wall: a vertical wall facing the camera with a V-shaped notch; the rays of rows v minus 1 and v plus 1 hit the flat wall with normals pointing straight at the camera, and the ray of row v hits the notch's upper face, its dot red, its normal pointing down and to the left; the verdict in red reads v is a spike, skipped, because v turns away from both neighbours while those two agree and lie on one plane, so v's two checks and the checks toward v are skipped. In the second, a step: a lower riser in front, a slanted top face, and a higher riser further back, both risers facing the camera; rows v plus 1 and v minus 1 hit the two risers with the same normal and row v hits the slanted top with a normal pointing up and left; a double arrow marks the tread between the risers; the verdict in green reads not a spike, kept, because the two risers face the same way but their planes are a tread apart. In the third, a wire in front of a wall: rows v minus 1 and v plus 1 hit the wall, row v hits a small round wire well in front of it; the verdict in red reads v is a thin foreground, skipped, because v is nearer than both neighbours, which are one surface, so both of v's silhouette checks are skipped. A caption defines one surface as normals within about 7 degrees and the far sample on the near one's plane within 0.3% of the depth](/assets/images/thin-outline/normal-spike.svg)

### One Surface

Two pixels are **one surface** for the filter when both of these hold:

- their normals agree, with the length of their cross product below 0.125, about 7°;
- **and** the second pixel's sample lies on the first one's plane, with its offset along the first normal within
  0.3% of the first pixel's depth.

```hlsl
bool IsSameSurfaceForSpike(FThinOutlineSample A, FThinOutlineSample B)
{
	return A.bHasNormal && B.bHasNormal
		&& length(cross(A.WorldNormal, B.WorldNormal)) < SpikeThreshold
		&& abs(dot(B.TranslatedWorldPosition - A.TranslatedWorldPosition, A.WorldNormal)) <= SpikePlaneTolerance * A.Depth;
}
```

#### Why the Plane Test Is Needed

The plane test came from the first trial. With the normals alone, the filter removed the long crease along the top of
the low walls in my test level.

Each wall top there is a thin step between the wall's front and the next wall behind it, both facing the camera, 220
and 230 units away. On the frames when the jitter put the whole top into one pixel row, that row had a crease toward
both neighbours, and the neighbours' normals agreed: a spike, by normals.

A groove's two sides lie on one plane. A step's two risers are a tread apart, here 4.5% of the depth, and the plane
test tells them apart. With it, the wall tops' records came back exactly as they were without the filter.

### What the Spike Filter Removes and Costs

**What it drops.** What the filter costs in pictures is a real ridge one pixel wide on a flat surface, such as a thin
wall seen edge-on at a distance. It is dropped with the spikes. That was my first post's intent for thin geometry
too.

**What it changes in my test level.** On one frame, it skips about 400 checks, all on the character's fine features:
the fingers, the face, the folds of the torso. It skips nothing on the level. Static, against the filter off, it
changes 48 display pixels by more than a tenth of full strength, 25 removed and 23 added, all on the character.

**The cost.** It costs about 0.002 ms at 1280×720 and 73%. Loading the far normal for every pixel cost 0.007 ms;
loading it only where a crease fired cut that to a third.

With the spike filter added to the presence:

<iframe style="width: 100%; aspect-ratio: 16 / 9; border: 0;" src="https://www.youtube-nocookie.com/embed/XQub3vIpPHk" title="City Sample, presence and spike filter on" allow="accelerometer; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

*The City Sample with the presence and the spike filter on.
[Watch on YouTube](https://youtu.be/XQub3vIpPHk).*

## The Thin Foreground

My first post argued for filtering thin geometry when it is sampled, rather than after the fit. By the time the dense
edge suppression runs, the thin geometry's samples have been mixed into the records around it and have skewed their
fits. A suppression can only hide an edge pixel; it cannot clean up its neighbours' records.

A foreground one pixel thin, such as a wire, a pole or a railing at a distance, is the depth version of the normal
spike. It is hit by the jittered sample on some frames, so it pops. And its two silhouettes are one pixel apart: an
isolated pair, which the dense suppression keeps by design.

### The Rule

The rule is the spike rule on depth. Take a pixel nearer than both of its neighbours along an axis, while those two
are one surface by the same test. That pixel records no silhouette toward either side. Its depth edge measures are
cleared too, so that the presence does not find it either.

Silhouette records are kept by the foreground pixels, as the
[silhouettes]({% post_url 2026-10-11-thin-outline-silhouettes %}) post explains. So the samples of a one-pixel
foreground are only ever its own: the pixels beside it are the background toward it, and record nothing. The near
neighbours therefore suffice, and no sample two pixels away is needed.

The pixels beside it keep their background checks, which the history uses. The fetch velocity of a background pixel
and the background-within-reach test still see the wire.

### What It Leaves Open and Costs

**The opposite case is left open:** a background gap one pixel wide between two foreground parts. The dense
suppression's both-sides trigger leaves it alone, as [The Trigger](#the-trigger) says, and this filter only looks at
foregrounds.

**The cost.** My test level has no thin foreground, so I have no number for this filter beyond its cost. It needs no
extra loads, since the near neighbours' normals are loaded anyway. Measured together with the spike filter, the two
cost 0.004 ms at 1280×720 and 73%.

<!-- capture (City Sample, wires or railings at a distance): thin-foreground filter off against on -->

## What Is Left

The outline still flickers in places:

- **Grooved surfaces at certain distances.** Moving the ramp's ends removed that flicker, but made other creases at
  the same distance dashed, and the ramp cannot be tuned by distance. The isolated edge suppression above is what I
  use there.
- **Junction creases.** A junction crease found on every other frame alternates between two faint strengths.
- **A stray silhouette over a crease.** The choice between the types still puts silhouettes first. So a stray
  silhouette record whose fit is valid can hide a converged crease for a frame. The presence dims that silhouette,
  but its fit still counts as an edge.

## Summary

The two suppressions run once per rendering pixel, on the edge the axis selection chose. They run after the spatial
filter and before the band's coverage is computed, the dense test first, since it is cheaper and decides more pixels.
The presence and the two spike filters act where the records are sampled and updated, so the records themselves are
cleaner.

Every step, in the order of this post:

| Step | Where | What it fixes | What it does not |
|---|---|---|---|
| Keeping a record through a missed contour (history reprojection post) | history | Silhouette records reset once per jitter cycle where the check misses the contour | Records whose surface is gone, dropped at once; records that pass every test |
| Dense edge suppression | drawing | Tangles of lines where three or more edges crowd within two pixels | The flicker of dense edges; isolated pairs, including one-pixel geometry |
| Isolated edge suppression | drawing | Specks of one or two pixels, among them what is left of the grooves | The flicker of lines; it removes 0.6% of the outline |
| Presence | records and drawing | Edges found on few recent frames: pops, junctions, ghosts | The footprints of a walking character, which fade too slowly (the velocity test) |
| Normal-spike filter | sampling | Groove walls thinner than a pixel, before they reach a record | Real one-pixel ridges, which it drops with them |
| Thin-foreground filter | sampling | Wires and poles one pixel thin | A one-pixel background gap |

Of these, the presence and the spike filter removed most of the flicker I saw in the City Sample, at a cost of a few
thousandths of a millisecond. What the whole outline costs, and how it is drawn from the records, is in the
[drawing after the upscaler]({% post_url 2026-10-11-thin-outline-drawing-after-upscaler %}) post.
