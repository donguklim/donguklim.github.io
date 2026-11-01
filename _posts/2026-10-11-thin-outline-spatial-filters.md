---
layout: post
title: "Thin Outlines From the G-Buffer Under TAA: Spatial Filters"
date: 2026-10-11 13:00:00 +0900
categories: [rendering, npr]
hidden: true
---

Each rendering pixel reconstructs its edge from its own pooled record, and nothing so far makes neighbouring pixels
agree with each other. This post covers the spatial filter that runs on the chosen edge before it is drawn. Blending
the neighbours' lines is the obvious thing to do, and the post shows why it fails on a static edge. Instead, the
filter refits the line through the pixels across the edge's axis: its slope, and since October 2026 its position too,
through a run of five pixels. The post also says what the filter is for and what it is not. It did not fix the
flicker, which has its own post, and it does not remove the beads of a thin band on a diagonal, which the drawing post
does. It is part of a
[series]({% post_url 2026-10-11-thin-outline %}) and assumes the
[core idea]({% post_url 2026-10-11-thin-outline-core-idea %}) post.

## The Seam Problem, and What Was Left of It

Each rendering pixel fits its own line from its own pooled record. So the outline is temporally stable within each
pixel, but not spatially stable across neighbouring pixels. Where a contour passes from one pixel into the next, the
two segments have slightly different slopes and intercepts, and don't quite meet.

In my first design, drawn before the upscaler, I saw the coverage along such seams shift as the records updated, and
the outline fade in and out there. My answer was a spatial filter in the spirit of SVGF (Spatiotemporal
Variance-Guided Filtering, Schied et al. 2017). It blended each pixel's line with the lines of its eight neighbours,
and weighted every neighbour by:

- its uncertainty;
- how far its line passed from the target's centre;
- how well the two lines' directions agreed;
- depth;
- how much its record mixed in a second edge.

It tracked the blend's uncertainty, so that a second pass could reach one pixel further along the edge.

Then drawing after the upscaler, described in
[its own post]({% post_url 2026-10-11-thin-outline-drawing-after-upscaler %}),
showed where the fading and the dashes along the seams came from: the upscaler's handling of a thin, jittered
outline, not the seams. Drawn at display resolution from the same records, they were gone.

What remained visible of the pixels' disagreement was something I hadn't noticed under the upscaler: the slope. Below
100% screen percentage, a vertical crease came out as a two-row sawtooth.

The rest of this post is about that sawtooth and the filter that removes it:

1. Along a static edge, every row shares the same slope error
   ([The Sawtooth](#the-sawtooth-every-row-shares-the-same-slope-error)).
2. Blending the neighbours' lines cannot remove a shared error ([Why Blending Lines Fails](#why-blending-lines-fails)).
3. The filter fits a line through the rows' centre crossings instead, gated by a collinearity test
   ([Positions, Not Lines](#positions-not-lines)).
4. The measurements show what the filter changed, on a static edge and on a diagonal one
   ([What the Filter Changed](#what-the-filter-changed)). The last section says what the filter is for
   ([What the Filter Is For](#what-the-filter-is-for)).

## The Sawtooth: Every Row Shares the Same Slope Error

The terms are those of the core idea post, for the horizontal axis. In rendering pixels from the pixel's top-left
corner, $u$ runs along the axis ($u = x$) and $v$ across it ($v = y$). The fitted edge is the line

$$
u = a + b\,v .
$$

The intercept $a$ is where the line crosses the pixel's top side. The slope $b$ is how far the edge moves along the
axis per pixel of travel across it.

One more quantity matters here: the **centre crossing**, the point where the line crosses the row through the pixel's
centre,

$$
u_c = a + \tfrac12\,b .
$$

The vertical axis is the same, with $x$ and $y$ swapped.

### One Static Crease

I looked at a vertical crease at the corner of a block in my test scene, in a static frame at 50% screen percentage
and 1280×720: rendering column 117, rows 199 to 206. Every row's pooled fit was the same line: $a = 0.744$,
$b = 0.37$, $u_c = 0.930$.

The real edge is vertical, so $b$ should be 0. The slope's standard error is about 0.26 (0.129 stored for the
record's midline, doubled like the decoded edge). So 0.37 is 1.4 standard errors from 0: ordinary noise for a slope
read off a single pixel of height. What is not ordinary is that every row has the *same* noise.

### The Same Noise in Every Row

The jitter is global: one offset per frame, the same for every pixel. A vertical edge sits at the same $u$ in every
row. So every row's record receives exactly the same $(u, v)$ samples, classified the same way, and computes exactly
the same fit, with exactly the same slope error.

Noise that differs from pixel to pixel would look like a rough line. Noise that is the same everywhere looks like a
regular pattern:

![Two columns of three square rendering pixels, labelled column 117 and 118 and rows 199 to 201, with a dashed green vertical line, the true edge, at u equals 0.93 inside the left column. In each row a red line leans from u equals 0.744 at the row's top to 1.114 at its bottom, the same in every row, so the red segments form a sawtooth that jumps back at every row boundary; a blue dot at the middle height of each row marks the centre crossing at 0.93, which lies on the green line in every row. A dimension arrow under the bottom row shows the lean of b equals 0.37 pixels per row, and dotted grey horizontal lines halve each row into the two display pixel rows of 50% screen percentage. The caption says the frame is static at 50% and 1280×720, and the axis is the horizontal one with u along it and v across it](/assets/images/thin-outline/shared-slope-sawtooth.svg)

Two things in that figure matter for everything that follows:

- **The lines lean**, all the same way, and jump back 0.37 pixels at every row boundary. At 50%, a rendering pixel row
  is two display rows, so the lean shows as a two-row sawtooth. Down display column 235, the drawn alpha alternated
  0.85 and 0.48; in the column to its right, 0.21 and 0.58.
- **The centre crossings** are all at $u = 0.930$, exactly on the true edge. Whatever error the crossings share only
  slides the whole edge sideways, by the same amount in every row, which is invisible.

### Why Drawing Before the Upscaler Hid It

Drawn before the upscaler, the sawtooth had been there too, but hidden:

- At 100%, the lean stays inside one display pixel.
- At lower percentages, each frame draws the alpha of the display pixel under the jittered sample. So on some frames
  the top half of the rendering pixel reports, and on others the bottom half, and the upscaler blends them over many
  frames.

Drawn after the upscaler, nothing averages it.

## Why Blending Lines Fails

### A Neighbour's Line in the Target's Frame

Take a target pixel $p$, with its neighbour $A$ directly above (the row with $v$ one less) and $B$ directly below. A
neighbour's line is brought into the target's frame by re-expressing it in $p$'s coordinates. For $A$, a point at
$p$-local $v$ is at $v_A = v + 1$, so $A$'s line $u = a + b v_A$ becomes $u = (a + b) + b v$.

On the static edge above, $a + b = 1.114$. At $p$'s centre row, $v = \tfrac12$, $A$'s line is at $u = 1.299$, while
$p$'s own crossing is at 0.930. The miss is exactly $b = 0.37$.

### Blending Keeps the Shared Error

My first filter's continuity term compared the neighbour's line with the target's at the target's centre, with a scale
of 0.25 pixels:

$$
w_{\text{continuity}} = \exp\!\left[-\left(\frac{0.37}{0.25}\right)^2\right] = 0.11 .
$$

So the two neighbours along the edge entered with about a tenth of the target's weight, and the filter nearly left the
line alone.

Worse, even with all three weights forced to 1, the blend of three lines that all have slope $b$ has slope $b$. The
offsets 1.299, 0.930 and 0.561 average back to 0.930, and the slope is 0.37 in every input. The shared error passes
straight through.

The filter's uncertainty bookkeeping was wrong for it too. The confidence term and the variance update assumed
independent errors, so that averaging three neighbours divides the variance by about three. With a shared error, the
variance after averaging is unchanged. The filter would have reported a tightening that did not happen.

### Comparing the Neighbours Doubles the Error

My improvement idea was to compare the neighbours with each other, instead of with the possibly unreliable target:
extend $A$'s line through the target to the far side, and see whether it meets $B$'s line.

On the static edge, $A$'s line at $B$'s centre row, $v = \tfrac32$, is at $u = 1.114 + 0.37 \times 1.5 = 1.669$, and
$B$'s crossing is at 0.930. The miss is $2b = 0.74$, twice the first filter's, because the extension travels two rows:

![Two columns of three square pixels labelled A above, p target and B below. A's fitted line is drawn solid red through A, from u equals 0.744 at A's top to 1.114 at its bottom, and continues dashed through p and B at the same lean. Blue dots mark the centre crossings of A, p and B, all at u equals 0.93. Small red dots mark where A's extended line crosses p's centre row, at 1.299, and B's centre row, at 1.669, with dimension arrows from the blue dots to them labelled b equals 0.37 and 2b equals 0.74. Faint red segments show p's and B's own fitted lines, identical to A's one row lower each. The legend says that line against line at the target's centre row misses by 0.37 pixels for a continuity weight of 0.11, and that the line extended through the target misses B's crossing by 0.74 pixels for a weight of 0.0002, so A is discarded; the gate scale is 0.25 pixels](/assets/images/thin-outline/line-extension.svg)

| What is compared | Miss on the static edge | Weight at a scale of 0.25 px |
|---|---|---|
| $A$'s line against $p$'s line, at $p$'s centre row | $b = 0.37$ px | 0.11 |
| $A$'s line extended to $B$'s centre row, against $B$'s line | $2b = 0.74$ px | 0.0002 |

The idea was meant for the case where the target's line is unreliable. But it relied instead on the neighbours'
*slopes*, the least reliable part of every record. When the slope error is shared, the far-side comparison doubles
it. So on a static edge, the test discards exactly the neighbours it was meant to accept.

Including $B$'s adjacent cells, as the idea proposed, does not help. Their pooled records describe the same edge and
report the same crossing in the target's frame.

## Positions, Not Lines

### The Reliable Quantity Is the Centre Crossing

The core idea post gives the standard error of the line's position at the pixel's centre. Let $s$ be the residual
standard deviation and $n$ the distinct sample count. The samples' mean $\mu_v$ is close to $\tfrac12$, because the
jitter covers the pixel evenly. Then

$$
\mathrm{SE}(u_c) \approx \frac{s}{\sqrt n}, \qquad
\mathrm{SE}(b) = \frac{s}{\sqrt{n\,\sigma_v^2}} .
$$

Inside one pixel, the samples spread uniformly over $[0, 1]$ across the axis, so $\sigma_v^2 \approx \tfrac1{12}$, and

$$
\mathrm{SE}(u_c) \approx \mathrm{SE}(b)\,\sigma_v \approx \frac{\mathrm{SE}(b)}{3.5} .
$$

In plain words: within one pixel, the samples span only one pixel of $v$. So the slope is read off a lever with a
very short arm, while the position at the centre is just a mean. In the example, that is 0.075 pixels against 0.26.

And for an OLS fit, the estimate of the line at the samples' mean $v$ is uncorrelated with the slope estimate. A slope
error tilts the line *around* the centre crossing without moving it. So each row's record gives:

- one trustworthy point, its centre crossing;
- one untrustworthy number, its slope.

The filter should use the points and ignore the slopes.

### The Points in the Target's Frame

In the target's frame, the three points are

$$
P_A = \left(u_A,\ -\tfrac12\right), \qquad
P_p = \left(u_p,\ \tfrac12\right), \qquad
P_B = \left(u_B,\ \tfrac32\right),
$$

where $u_q$ is the centre crossing of $q$'s **pooled** fit of the drawn edge's type. Each row's fit is pooled the way
that pixel pools it:

- for a crease, its own record and its two neighbours' along the axis;
- for a silhouette, the pool of the drawn edge's background side, since the rows above and below hold the same
  contour's records on the same side.

The cells one step along the axis, which my idea's "B's adjacent cells" were for, need no separate treatment. Each
row's pool already merges the three cells of that row. And the selected axis has $\lvert b \rvert \le 1$ (selection
prefers the axis the edge crosses squarely), so the edge in the next row lies within that pool's reach.

### The Collinearity Test

Do $A$ and $B$ coincide with each other through the target? That is the question whether the three points are
collinear. The slope-free statistic for it is the second difference

$$
d = \frac{u_A + u_B}{2} - u_p ,
$$

the distance between $p$'s crossing and the midpoint of $A$'s and $B$'s. No slope appears in it, so the shared slope
error cannot enter.

| Situation | $d$ |
|---|---|
| One straight edge through all three rows, any slope | 0 |
| The same edge in $A$ and $p$, a parallel edge one pixel over in $B$ | $\pm 0.5$ |
| A corner at $p$ between slopes $b_1$ and $b_2$ | about $(b_1 - b_2)/2$ |
| A curved edge of radius $R$ pixels | about $1/(2R)$: 0.05 at $R = 10$, 0.25 at $R = 2$ |

![Three panels of two columns by three rows of square pixels, with blue dots for the centre crossings of the rows' pooled fits at the row centres. In the first panel, a static vertical edge, the three dots line up at u equals 0.93, a blue vertical line is fitted through them, faint red leaning segments show the old fitted lines, and the caption says d equals 0 and the fitted slope is 0, so the sawtooth is gone. In the second panel, a slanted edge of slope 0.5, the dots lie at 0.43, 0.93 and 1.43 and a blue line of slope 0.5 passes through all three; the caption says d is 0 and the fitted slope is 0.5, and that no slope enters d so any straight edge passes. In the third panel the top two dots lie on a dashed green edge at 0.93 and the bottom row holds a red dot on a different dashed green edge at 1.93, a parallel edge one pixel over; the caption says d equals 0.5, the gate weight is about 0.02, B is left out and the own slope is kept](/assets/images/thin-outline/collinearity-cases.svg)

The gate on the pair is

$$
g = \exp\!\left[-\left(\frac{d}{\sigma_c}\right)^2\right],
\qquad \sigma_c = 0.25 \text{ rendering pixels}.
$$

Besides the gate, the neighbours must be of the drawn edge's type, with a valid fit. Two gates of my first filter are
not needed:

- **No depth gate**, for creases or silhouettes. A neighbour row on another surface or object can only pass the gate
  when its crossing is collinear with the other two. And then the slope through it is the slope through them.
- **No direction gate.** It compared slopes, and the second difference already stops corners.

### The Co-Trigger Weight, Tried and Dropped

The last of the first filter's weights, for a record that mixes in a second edge, did go into the plugin at first.

The pooling of the core idea post assumes that a single edge crosses the pixel along the axis. With two edges less
than a pixel apart, a jittered sample that lands on the sliver between them fires *both* checks of the axis in one
frame, the left and the right one. I called that a co-trigger. Every record kept a decayed count of its co-trigger
frames next to its sample count. The **co-trigger ratio** was the share of a record's frames that were co-triggered:
0 for a clean edge, approaching 1 for a pixel that saw two edges on most frames.

The ratio was a reliability signal. My first design used it to leave co-triggered records out of the fit altogether.
As the core idea post tells, that produced dashed outlines, and it was turned off.

The filter used the ratio more gently. Each neighbour row's weight was multiplied by one minus its pooled record's
ratio, so that a row mixing in a second edge counted less in the slope. With the weight on and off, the filtered
slopes showed no significant difference. That fits the argument above: such a row can only pass the collinearity gate
when its crossing lines up with the other two anyway. So I dropped the weight, and then the ratio itself. Its channel
in the record now holds the presence of the
[flicker suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %})
post.

### The Weights

The fit takes one weight per point, and one for the pixel's own slope, which is kept as a prior. Each point is
weighted by the inverse variance of its centre crossing: the standard error of the core idea post, computed from that
row's pooled fit. The two neighbours' weights are also multiplied by the gate:

$$
w_A = \frac{g}{\mathrm{SE}(u_A)^2}, \qquad
w_p = \frac{1}{\mathrm{SE}(u_p)^2}, \qquad
w_B = \frac{g}{\mathrm{SE}(u_B)^2}, \qquad
w_b = \frac{1}{\mathrm{SE}(b_p)^2} .
$$

**Inverse variance.** Inverse-variance weights are the minimum-variance way to combine estimates of the same quantity.
So a row whose crossing is well known, with many distinct samples and little scatter, pulls the line harder than a row
whose record was just reset.

**The gate on the neighbours only.** The target's point is the one being filtered, and is never in doubt. The
neighbours are judged as a pair by the second difference, so when $d$ is large, both fall out together.

**The own slope.** Its weight $w_b$ is the inverse variance of the pixel's own slope, with the standard error of the
decoded edge, twice the stored record line's.

**A floor on the standard errors.** Every standard error is floored at 0.001 pixels before the division. So a perfect
fit of zero residual, which two distinct jitter positions produce, gets a large but finite weight instead of swamping
the rest.

### The Run Across the Axis

The run does not stop at the two neighbours. The pixels two rows away, $A_2$ at $v = -1.5$ and $B_2$ at $v = 2.5$,
join the fit as a second pair:

- with a gate of their own, $g_2 = \exp(-(d_2/\sigma_c)^2)$, where $d_2$ is the distance between $p$'s crossing and the
  midpoint of theirs; like $d$, it is zero on any straight edge and grows at a corner;
- with the same inverse-variance weights.

A pair with one member missing adds nothing: a one-sided far point could only be judged with the own slope. The run is
three, five or seven pixels across the axis, five by default.

If a neighbour at distance one is missing on either side, at an edge end or a corner, the own fit is kept as it is. A
one-sided test cannot tell a slanted edge from a parallel one without using the own slope, the very thing the filter
distrusts. And edge ends are rare.

### The Fitted Slope

With these weights, the slope minimizes

$$
\sum_{q \in \{A, p, B\}} w_q\,(u_q - a - b\,v_q)^2 + w_b\,(b - b_p)^2 .
$$

This is a weighted least-squares line through the points of the run, with a penalty for leaving the own slope. The sum
is written for three rows; when all five points are in, it runs over $A_2, A, p, B, B_2$. With the weighted means
$\bar v = \sum_q w_q v_q / \sum_q w_q$ and $\bar u = \sum_q w_q u_q / \sum_q w_q$, the solution is

$$
\hat b = \frac{S_{vu} + w_b\,b_p}{S_{vv} + w_b}, \qquad
S_{vv} = \sum_q w_q\,(v_q - \bar v)^2, \quad
S_{vu} = \sum_q w_q\,(v_q - \bar v)(u_q - \bar u) .
$$

#### The Own Slope as a Prior

The prior is there for a graceful fallback. Without it, the target's own point sits at $\bar v$ and contributes
nothing to the slope. Then any nonzero gate, however small, would hand the slope entirely to the neighbours. With the
prior, the slope is the inverse-variance blend of the crossings' slope and the own slope, and it tends to the own
slope as the gate closes.

On the static edge, all three crossings have a standard error of 0.075 pixels, and the gate is 1. So:

- each point weighs $1/0.075^2 = 178$;
- $\bar v = \tfrac12$, and $S_{vv} = 178 \times (1 + 0 + 1) = 356$;
- the own slope weighs $1/0.26^2 = 15$.

The prior's share is $15 / (356 + 15)$, about 4%. The section
[Why the Within-Pixel Slopes Get Almost No Weight](#why-the-within-pixel-slopes-get-almost-no-weight) arrives at the
same figure from the other direction. On a record with few distinct samples, whose slope standard error is large, the
prior is nothing. On a perfect fit, whose standard error of 0 is floored at 0.001 pixels, it is everything.

#### In the Shader

In HLSL, the points go into running sums, one pair at a time, each pair behind its own gate:

```hlsl
void AddCrossing(inout FThinOutlineCrossingSums Sums, float Weight, float V, float U)
{
	Sums.W += Weight;
	Sums.Wv += Weight * V;
	Sums.Wu += Weight * U;
	Sums.Wvv += Weight * V * V;
	Sums.Wvu += Weight * V * U;
}

// The pair of across fits at distance Across (Before at -Across, After at +Across), gated together.
void AddCrossingPair(inout FThinOutlineCrossingSums Sums, FThinOutlineEdgeFit Before, FThinOutlineEdgeFit After, float Across, float OwnCrossing)
{
	if (!Before.bValid || !After.bValid)
	{
		return;
	}
	// The neighbours' centre crossings are in this pixel's coordinates already; they sit at v = 0.5 -/+ Across.
	const float BeforeCrossing = Before.Intercept + 0.5 * Before.Slope;
	const float AfterCrossing = After.Intercept + 0.5 * After.Slope;
	const float Collinearity = 0.5 * (BeforeCrossing + AfterCrossing) - OwnCrossing;
	const float Gate = exp(-Pow2(Collinearity / SpatialFilterSigma));
	AddCrossing(Sums, Gate / Pow2(max(Before.CrossingStandardError, THIN_OUTLINE_MIN_STANDARD_ERROR)), 0.5 - Across, BeforeCrossing);
	AddCrossing(Sums, Gate / Pow2(max(After.CrossingStandardError, THIN_OUTLINE_MIN_STANDARD_ERROR)), 0.5 + Across, AfterCrossing);
}
```

Once the own point and the pairs at distance one and two are in, the shader solves for the line. Both neighbours at
distance one must be valid; otherwise nothing happens:

```hlsl
// The edge line's slope has twice the record line's standard error.
const float SlopeWeight = 1.0 / Pow2(max(2.0 * Own.SlopeStandardError, THIN_OUTLINE_MIN_STANDARD_ERROR));

const float MeanV = Sums.Wv / Sums.W;
const float MeanU = Sums.Wu / Sums.W;
const float Svv = Sums.Wvv - MeanV * Sums.Wv;
const float Svu = Sums.Wvu - MeanV * Sums.Wu;
const float Slope = (Svu + SlopeWeight * Own.Slope) / (Svv + SlopeWeight);
const float Crossing = MeanU + Slope * (0.5 - MeanV);

Candidate.Fit.Slope = Slope;
Candidate.Fit.Intercept = Crossing - 0.5 * Slope;
```

With equal weights and only the first pair accepted, $\hat b = (u_B - u_A)/2$. On the static edge, that is 0, and the
sawtooth is gone.

#### Moving the Crossing Onto the Line

The last two lines of the shader move the centre crossing too. The pixel's position becomes the fitted line's value at
its own centre, $\hat u_p = \bar u + \hat b\,(\tfrac12 - \bar v)$. That is the inverse-variance blend of its own
crossing with where the neighbours say the line runs.

For the first version of the filter, I kept the own crossing. I reasoned that a position change is a fraction of a
pixel, which coverage painting turns into a few percent of thickness.

A diagonal edge showed what that fraction does. Along an 18° crease, 1,813 pixels long, the crossings scattered 0.041
pixels rms about one straight line. After the upscaler at 50%, each rendering pixel paints its own line, clipped to its
own square. So that scatter showed as a small step of the band at every rendering pixel's border, and the band's
centre wandered by ±0.17 display pixels. Moving the crossing onto the line through five brought the scatter to 0.015
pixels and the wander to ±0.06.

The dense edge suppression of the
[flicker suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %})
post compares the crossings the filter produces. They move by a few hundredths of a pixel, well inside its tolerance.

### Why the Within-Pixel Slopes Get Almost No Weight

#### Under Independent Errors

Suppose for a moment that the rows' errors were independent. The slope from the two outer crossings then has variance

$$
\mathrm{Var}(\hat b_{\text{rows}}) = \frac{\mathrm{SE}(u_A)^2 + \mathrm{SE}(u_B)^2}{4} = \frac{2 \times 0.075^2}{4} = 0.053^2 ,
$$

against $0.26^2$ for the target's own slope. Inverse-variance weighting, the minimum-variance combination of
independent estimates, gives the within-pixel slope a weight of

$$
\frac{0.053^2}{0.26^2 + 0.053^2} = 4\% .
$$

#### Under the Shared Error

Now drop the independence assumption, which the static edge violates: the rows' errors are shared. A shared error in
the crossings moves all three points by the same amount. The fit absorbs it into the position and leaves $\hat b$
untouched. So on the static edge, the slope through the crossings is *exact*. The within-pixel slope is wrong by 0.37
in every row, and averaging rows does not help it.

The right weight of every within-pixel slope, the target's and the neighbours', is therefore about zero whenever both
neighbours are accepted: 4% at most under independence, nothing under the shared error. The rule "slope from the
crossings, fall back to the own slope as the gate closes" is as close to optimal as three rows allow.

One pass is enough, too. The iterations of my first filter existed to reach farther along the edge for line
averaging, which the shared error makes pointless.

#### A Longer Run

A longer run across the axis is another matter. On a slanted edge, the rows' crossing errors are not shared, and then
more points do help. The slope through crossings one row apart has variance $\mathrm{SE}(u)^2 / S_{vv}$, with
$S_{vv} = \sum_q (v_q - \bar v)^2$. On the 18° crease, the own slopes ranged from 0.10 to 0.88 for a true slope of
0.32. Per length of the run:

| Rows in the run | $S_{vv}$ | Slope error on the 18° crease, rms |
|---:|---:|---:|
| 3 | 2 | 0.042 |
| 5 | 10 | 0.012 |
| 7 | 28 | 0.007 |

So five rows make the slope $\sqrt 5$ times steadier than three, and seven only another $\sqrt{2.8}$. Five is the
default, since seven costs two more pooled fits for little.

One pass of five is also better than two passes of three. A second pass would refit through the crossings the first
produced:

- with slopes only, that is the same input twice;
- with positions, it becomes a smoothing that spreads each correction one row further, through an extra pass, instead
  of one line through five measured points.

### Pooling Across the Axis Instead

The same idea can be applied one level down, to the records. Pool the rows above and below as records shifted by
$\pm 1$ in $v$, and fit the pool. Take three identical records shifted by $-1$, 0 and $+1$. Pooling them leaves the
covariance unchanged, since each shift is a constant per copy. It raises the spread across the axis from
$\sigma_v^2 = \tfrac1{12}$ to $\tfrac1{12} + \tfrac23 = \tfrac34$:

$$
b_{\text{pooled}} = \frac{\sigma_{uv}}{\sigma_v^2 + \tfrac23}
= b_{\text{within}} \times \frac{1/12}{3/4} = \frac{b_{\text{within}}}{9} .
$$

On the static edge, the lean drops from 0.37 to 0.04 pixels, which is invisible. On a slanted edge, the between-row
differences supply the slope with $\tfrac89$ of the weight. So it would fix the sawtooth too. Its drawbacks decided
against it:

- there is no gate against a parallel edge or a corner in the next row, short of fitting that row anyway;
- it changes the fit that the axis selection and the band test depend on;
- it triple-counts samples in the standard errors.

With three rows, as first built, the fit-level version costs the same record loads: three pooled fits per axis instead
of one (the run of five adds two more). It keeps the selection untouched, and it gates explicitly.

## What the Filter Changed

### On the Static Edge

Down the sawtooth column of the static frame, the alpha of the horizontal axis's edges before and after the filter:

| Display rows | Column 235, before | after | Column 236, before | after |
|---|---|---|---|---|
| even (398, 400, ...) | 0.85 | 0.64 | 0.21 | 0.35 |
| odd (399, 401, ...) | 0.48 | 0.62 | 0.58 | 0.37 |

Rendering column 117's slope went from 0.370 to 0.014, and its centre crossing stayed at 0.930. The crops below are
from the final image: display columns 225 to 246 and rows 390 to 415 of the 50% frame, zoomed eight times. The filter
is off on the left and on on the right:

![Two side-by-side crops of the same purple block corner, each 22 by 26 display pixels zoomed eight times and separated by a magenta bar. On the left the dark vertical outline running down the block's edge is a staircase of alternating darker and lighter pixel pairs, a two-row sawtooth; on the right the same outline is a straight, evenly dark column](/assets/images/thin-outline/sawtooth-filter-off-on.png)

### Creases and Silhouettes

I first built the filter for creases only. After looking at zoomed screenshots, where the change is hard to see at a
glance and clear pixel by pixel, I had it applied to silhouettes too. On a 50% frame with both filtered, a crease pixel
was filtered when both its neighbour rows had valid crease fits. The counts and the slope changes:

| On the 50% frame | Drawn pixels filtered | Median slope change (px per px) | Pixels whose slope changed by more than 0.5 |
|---|---|---|---|
| Creases | 3,177 of 3,615 | 0.11 | 237 |
| Silhouettes | 4,443 of 4,570 | 0.07 (90th percentile 0.22) | 8 |

- **Creases.** The median change is the shared error of the static edges. The crease pixels above 0.5 all had an own
  slope standard error above 3 (checked on the first 50% frame, filtered for creases only, where they were 235). They
  were records with few distinct samples, whose own slope was noise, on edges whose neighbour crossings agreed, so the
  filter straightened them.
- **Silhouettes moved less.** Their records are the better converged ones, since the foreground keeps them and the
  kept ratio decides the side. So their own slopes were nearer the neighbours' to begin with.

### Cost and Motion

The cost, at 1280×720 on an RTX 5080 over a static view, as the median of 600-frame profiles:

| Drawing mode | Filter off | Filter on |
|---|---|---|
| before the upscaler, 100% | 0.137 ms | 0.148 ms (+8%) |
| after the upscaler, 50% | 0.076 ms | 0.085 ms (+12%) |

Silhouettes cost more than creases. Half of the drawn pixels are silhouettes, and a silhouette neighbour row pools up
to four records, with their kept ratios; a crease row pools three, without.

Under motion, the neighbour rows' fits come from the same reprojected records as the pixel's own, so nothing new
moves. A disocclusion resets rows unevenly, which the gate and the inverse-variance weights are there to cover.

### On a Diagonal Edge

The run of five and the moved crossing came from a diagonal edge. I rotated a box so that its creases and silhouettes
ran at 18° on screen, and the drawn outline showed a small staircase. The staircase turned out to be mostly the
painting of a thin band, which the
[drawing after the upscaler]({% post_url 2026-10-11-thin-outline-drawing-after-upscaler %})
post takes up. What the filter could still improve, I measured on that edge with the Python mirror of the edge pass,
on a converged capture at 50% after the upscaler:

| Along the 18° edge | Three rows, slope only | Five rows, slope and position |
|---|---|---|
| Crossing scatter about the straight line, rms | 0.041 px | 0.015 px |
| Slope error, rms | 0.042 | 0.012 |
| Wander of the band's centre, 10th to 90th percentile | ±0.17 display px | ±0.06 display px |
| Frame-to-frame spread of a pixel's crossing, median over ten frames | 0.010 px | 0.004 px |
| Frame-to-frame spread of a pixel's slope | 0.009 | 0.002 |

Two findings show where the gain came from:

- Every pixel of that edge chose the same axis, so the axis selection had no part in the stair.
- Before the change, the crossings sat on one straight line to 0.04 pixels. The filter was already doing what three
  rows allow.

So the gain came from the two extra rows and from moving the position. The longer run and the position cost two more
pooled fits per drawn pixel: over the first version, +6% of the outline's time after the upscaler at 50%, and +4%
before it at 100%.

## What the Filter Is For

The filter runs once per rendering pixel, on the edge the axis selection chose, before the band's coverage is
computed. It is the first of three steps there. The two suppressions of the
[flicker suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %})
post follow it:

- the dense one compares the centre crossings the filter produces;
- the isolated one reuses the two nearest neighbour fits the filter loads.

| What it fixes | What it does not |
|---|---|
| The shared slope error: the sawtooth below 100% screen percentage after the upscaler, and the noisy slopes of records with few distinct samples; the scatter of the positions along a straight edge, within the run of five | The beads of a band thinner than two pixels on a diagonal, which are the painting's, not the lines'; and the flicker |

The filter was my first attempt at the fading along the seams, which drawing after the upscaler then removed on its
own. It did nothing for the flicker that remained, which the flicker post is about.

The other posts have the rest:

- the [drawing after the upscaler]({% post_url 2026-10-11-thin-outline-drawing-after-upscaler %}) post has the pass
  the filter runs in;
- the [tile classification]({% post_url 2026-10-11-thin-outline-tile-classification %}) post, how that pass runs only
  where the records are;
- the [core idea]({% post_url 2026-10-11-thin-outline-core-idea %}) post, the standard errors it weighs by.
