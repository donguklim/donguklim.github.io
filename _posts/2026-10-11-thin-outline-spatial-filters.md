---
layout: post
title: "Thin Outlines From the G-Buffer Under TAA: Spatial Filters"
date: 2026-10-11 13:00:00 +0900
categories: [rendering, npr]
hidden: true
---

Each rendering pixel reconstructs its edge from its own pooled record, and
nothing so far makes neighbouring pixels agree with each other. This post
covers the three steps that run on the chosen edge before it is drawn: a
spatial filter that straightens the line through the pixels across the
edge's axis, and two suppressions that leave an edge undrawn where other
edges crowd it, or where it runs over too few pixels to be a line. It also
says what each step is for and what it is not: none of them fixed the
flicker, which has its own post. It is part of a
[series]({% post_url 2026-10-11-thin-outline-plugin %}) and assumes the
[core idea]({% post_url 2026-10-11-thin-outline-core-idea %}) post.

## The Seam Problem, and What Was Left of It

Each rendering pixel fits its own line from its own pooled record, so the
outline is temporally stable within each pixel but not spatially stable
across neighbouring pixels. Where a contour passes from one pixel into the
next, the two segments have slightly different slopes and intercepts and
don't quite meet. In my first design, drawn before the upscaler, I saw the
coverage along such seams shift as the records updated, and the outline fade
in and out there. My answer was a spatial filter in the spirit of SVGF
(Spatiotemporal Variance-Guided Filtering, Schied et al. 2017): each pixel's
line blended with the lines of its eight neighbours, every neighbour
weighted by its uncertainty, by how far its line passed from the target's
centre, by how well the two lines' directions agreed, and by depth, with the
blend's uncertainty tracked so that a second pass could reach one pixel
further along the edge.

Then drawing after the upscaler, described in the
[drawing and performance]({% post_url 2026-10-11-thin-outline-drawing-and-performance %})
post, showed that the fading and the dashes along the seams came from the
upscaler's handling of a thin, jittered outline, not from the seams. Drawn at
display resolution from the same records, they were gone. What remained
visible of the pixels' disagreement was something I hadn't noticed under the
upscaler: the slope. Below 100% screen percentage a vertical crease came out
as a two-row sawtooth.

## The Sawtooth: Every Row Shares the Same Slope Error

The terms, for the horizontal axis as in the core idea post: $u$ runs along
the axis ($u = x$) and $v$ across it ($v = y$), in rendering pixels, from
the pixel's top-left corner. The fitted edge is the line

$$
u = a + b\,v ,
$$

with the intercept $a$ where the line crosses the pixel's top side and the
slope $b$ how far the edge moves along the axis per pixel of travel across
it. One more quantity matters here, the **centre crossing**

$$
u_c = a + \tfrac12\,b ,
$$

the point where the line crosses the row through the pixel's centre. The
vertical axis is the same with $x$ and $y$ swapped.

In a static frame at 50% screen percentage and 1280×720, I looked at a
vertical crease at the corner of a block in my test scene, rendering column
117, rows 199 to 206. Every row's pooled fit was the same line: $a = 0.744$,
$b = 0.37$, $u_c = 0.930$. The real edge is vertical, so $b$ should be 0.
The slope's standard error is about 0.26 (0.129 stored for the record's
midline, doubled like the decoded edge), so 0.37 is 1.4 standard errors from
0: ordinary noise for a slope read off a single pixel of height. What is not
ordinary is that every row has the *same* noise.

The jitter is global: one offset per frame, the same for every pixel. A
vertical edge sits at the same $u$ in every row, so every row's record
receives exactly the same $(u, v)$ samples, classified the same way, and
computes exactly the same fit, with exactly the same slope error. Noise that
differs from pixel to pixel would look like a rough line. Noise that is the
same everywhere looks like a regular pattern:

![Two columns of three square rendering pixels, labelled column 117 and 118 and rows 199 to 201, with a dashed green vertical line, the true edge, at u equals 0.93 inside the left column. In each row a red line leans from u equals 0.744 at the row's top to 1.114 at its bottom, the same in every row, so the red segments form a sawtooth that jumps back at every row boundary; a blue dot at the middle height of each row marks the centre crossing at 0.93, which lies on the green line in every row. A dimension arrow under the bottom row shows the lean of b equals 0.37 pixels per row, and dotted grey horizontal lines halve each row into the two display pixel rows of 50% screen percentage. The caption says the frame is static at 50% and 1280×720, and the axis is the horizontal one with u along it and v across it](/assets/images/thin-outline/shared-slope-sawtooth.svg)

Two things in that figure matter for everything that follows. The *lines*
lean, all the same way, and jump back 0.37 pixels at every row boundary. At
50% a rendering pixel row is two display rows, so the lean shows as a
two-row sawtooth; down display column 235 the drawn alpha alternated 0.85
and 0.48, and in the column to its right 0.21 and 0.58. And the *centre
crossings* are all at $u = 0.930$, exactly on the true edge. Whatever error
the crossings share only slides the whole edge sideways by the same amount
in every row, which is invisible.

Before the upscaler the sawtooth had been there too, and hidden: at 100% the
lean stays inside one display pixel, and at lower percentages each frame
draws the alpha of the display pixel under the jittered sample, so on some
frames the top half and on others the bottom half of the rendering pixel
reports, and the upscaler blends them over many frames. Drawn after the
upscaler nothing averages it.

## Why Blending Lines Fails

Take a target pixel $p$ with its neighbour $A$ directly above (the row with
$v$ one less) and $B$ directly below. A neighbour's line is brought into the
target's frame by re-expressing it in $p$'s coordinates: for $A$, a point at
$p$-local $v$ is at $v_A = v + 1$, so $A$'s line $u = a + b v_A$ becomes
$u = (a + b) + b v$. On the static edge above, $a + b = 1.114$, and at
$p$'s centre row, $v = \tfrac12$, $A$'s line is at $u = 1.299$, while $p$'s
own crossing is at 0.930. The miss is exactly $b = 0.37$.

My first filter's continuity term compared the neighbour's line with the
target's at the target's centre, with a scale of 0.25 pixels:

$$
w_{\text{continuity}} = \exp\!\left[-\left(\frac{0.37}{0.25}\right)^2\right] = 0.11 .
$$

So the two neighbours along the edge entered with about a tenth of the
target's weight, and the filter nearly left the line alone. Worse, even with
all three weights forced to 1, the blend of three lines that all have slope
$b$ has slope $b$: the offsets 1.299, 0.930 and 0.561 average back to 0.930,
and the slope is 0.37 in every input. The shared error passes straight
through. The filter's uncertainty bookkeeping was wrong for it too: the
confidence term and the variance update assumed independent errors, so that
averaging three neighbours divides the variance by about three. With a
shared error the variance after averaging is unchanged, and the filter would
have reported a tightening that did not happen.

My improvement idea was to compare the neighbours with each other instead of
with the possibly unreliable target: extend $A$'s line through the target to
the far side and see whether it meets $B$'s line. On the static edge, $A$'s
line at $B$'s centre row, $v = \tfrac32$, is at
$u = 1.114 + 0.37 \times 1.5 = 1.669$, and $B$'s crossing is at 0.930. The
miss is $2b = 0.74$, twice the first filter's, because the extension travels
two rows:

![Two columns of three square pixels labelled A above, p target and B below. A's fitted line is drawn solid red through A, from u equals 0.744 at A's top to 1.114 at its bottom, and continues dashed through p and B at the same lean. Blue dots mark the centre crossings of A, p and B, all at u equals 0.93. Small red dots mark where A's extended line crosses p's centre row, at 1.299, and B's centre row, at 1.669, with dimension arrows from the blue dots to them labelled b equals 0.37 and 2b equals 0.74. Faint red segments show p's and B's own fitted lines, identical to A's one row lower each. The legend says that line against line at the target's centre row misses by 0.37 pixels for a continuity weight of 0.11, and that the line extended through the target misses B's crossing by 0.74 pixels for a weight of 0.0002, so A is discarded; the gate scale is 0.25 pixels](/assets/images/thin-outline/line-extension.svg)

| What is compared | Miss on the static edge | Weight at a scale of 0.25 px |
|---|---|---|
| $A$'s line against $p$'s line, at $p$'s centre row | $b = 0.37$ px | 0.11 |
| $A$'s line extended to $B$'s centre row, against $B$'s line | $2b = 0.74$ px | 0.0002 |

The idea was meant for the case where the target's line is unreliable. But
what it relied on instead, the neighbours' *slopes*, is the least reliable
part of every record, and when the slope error is shared, the far-side
comparison doubles it. On a static edge the test discards exactly the
neighbours it was meant to accept. Including $B$'s adjacent cells, as the
idea proposed, does not help: their pooled records describe the same edge
and report the same crossing in the target's frame.

## Positions, Not Lines

### The Reliable Quantity Is the Centre Crossing

The core idea post gives the standard error of the line's position at the
pixel's centre. With $s$ the residual standard deviation and $n$ the distinct
sample count, and the samples' mean $\mu_v$ close to $\tfrac12$ because the
jitter covers the pixel evenly,

$$
\mathrm{SE}(u_c) \approx \frac{s}{\sqrt n}, \qquad
\mathrm{SE}(b) = \frac{s}{\sqrt{n\,\sigma_v^2}} ,
$$

and since the samples' spread across the axis inside one pixel is
$\sigma_v^2 \approx \tfrac1{12}$, uniform on $[0, 1]$,

$$
\mathrm{SE}(u_c) \approx \mathrm{SE}(b)\,\sigma_v \approx \frac{\mathrm{SE}(b)}{3.5} .
$$

In plain words: within one pixel the samples span only one pixel of $v$, so
the slope is read off a lever with a very short arm, while the position at
the centre is just a mean. In the example, 0.075 pixels against 0.26. And
for an OLS fit the estimate of the line at the samples' mean $v$ is
uncorrelated with the slope estimate: a slope error tilts the line *around*
the centre crossing without moving it. So each row's record gives one
trustworthy point, its centre crossing, and one untrustworthy number, its
slope. The filter should use the points and ignore the slopes.

In the target's frame the three points are

$$
P_A = \left(u_A,\ -\tfrac12\right), \qquad
P_p = \left(u_p,\ \tfrac12\right), \qquad
P_B = \left(u_B,\ \tfrac32\right),
$$

where $u_q$ is the centre crossing of $q$'s **pooled** fit of the drawn
edge's type, pooled as that pixel pools it: for a crease its own record and
its two neighbours' along the axis; for a silhouette the pool of the drawn
edge's background side, since the rows above and below hold the same
contour's records on the same side. The cells one step along the axis need
no separate treatment, which is what my idea's "B's adjacent cells" were
for: each row's pool already merges the three cells of that row, and the
selected axis has $\lvert b \rvert \le 1$ (selection prefers the axis the
edge crosses squarely), so the edge in the next row lies within that pool's
reach.

### The Collinearity Test

Whether $A$ and $B$ coincide with each other through the target is the
question whether the three points are collinear. The slope-free statistic
is the second difference

$$
d = \frac{u_A + u_B}{2} - u_p ,
$$

the distance between $p$'s crossing and the midpoint of $A$'s and $B$'s. No
slope appears, so the shared slope error cannot enter.

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

Besides it, the neighbours must be of the drawn edge's type with a valid
fit. No depth gate is needed, for creases or silhouettes: a neighbour row on
another surface or object can only pass the gate when its crossing is
collinear with the other two, and then the slope through it is the slope
through them. The direction gate of my first filter is not needed either; it
compared slopes, and the second difference already stops corners.

### The Fitted Slope

The slope is then a weighted least-squares line through the three points,
each weighted by the inverse variance of its crossing and the neighbours'
additionally by the gate, with the pixel's own slope kept as a prior
weighted by its own inverse variance:

$$
\hat b = \frac{S_{vu} + w_b\,b_p}{S_{vv} + w_b}, \qquad
S_{vv} = \sum_q w_q\,(v_q - \bar v)^2, \quad
S_{vu} = \sum_q w_q\,(v_q - \bar v)(u_q - \bar u),
$$

over $q \in \{A, p, B\}$, with $w_A = g / \mathrm{SE}(u_A)^2$,
$w_p = 1 / \mathrm{SE}(u_p)^2$, $w_B = g / \mathrm{SE}(u_B)^2$ and
$w_b = 1 / \mathrm{SE}(b_p)^2$, the weighted means $\bar v$ and $\bar u$
taken with the three $w_q$. The prior is there for a graceful fallback.
Without it the target's own point sits at $\bar v$ and contributes nothing
to the slope, so any nonzero gate, however small, would hand the slope
entirely to the neighbours. With it, the slope is the inverse-variance blend
of the crossings' slope and the own slope, and tends to the own slope as
the gate closes. On a converged static edge the prior's share is about 4%
(next section); on a record with few distinct samples, whose slope standard
error is large, it is nothing; and on a perfect fit, whose standard error of
0 is floored at 0.001 pixels, it is everything. In HLSL, once both
neighbours have a valid fit:

```hlsl
// The neighbours' centre crossings are in this pixel's coordinates already (no offset along the axis); they sit at
// v = -0.5 and v = 1.5.
const float BeforeCrossing = Before.Intercept + 0.5 * Before.Slope;
const float AfterCrossing = After.Intercept + 0.5 * After.Slope;
const float Collinearity = 0.5 * (BeforeCrossing + AfterCrossing) - OwnCrossing;
const float Gate = exp(-Pow2(Collinearity / SpatialFilterSigma));

const float BeforeWeight = Gate / Pow2(max(Before.CrossingStandardError, THIN_OUTLINE_MIN_STANDARD_ERROR));
const float OwnWeight = 1.0 / Pow2(max(Own.CrossingStandardError, THIN_OUTLINE_MIN_STANDARD_ERROR));
const float AfterWeight = Gate / Pow2(max(After.CrossingStandardError, THIN_OUTLINE_MIN_STANDARD_ERROR));
// The edge line's slope has twice the record line's standard error.
const float SlopeWeight = 1.0 / Pow2(max(2.0 * Own.SlopeStandardError, THIN_OUTLINE_MIN_STANDARD_ERROR));

const float TotalWeight = BeforeWeight + OwnWeight + AfterWeight;
const float MeanV = (-0.5 * BeforeWeight + 0.5 * OwnWeight + 1.5 * AfterWeight) / TotalWeight;
const float MeanU = (BeforeWeight * BeforeCrossing + OwnWeight * OwnCrossing + AfterWeight * AfterCrossing) / TotalWeight;
const float3 DeltaV = float3(-0.5, 0.5, 1.5) - MeanV;
const float3 DeltaU = float3(BeforeCrossing, OwnCrossing, AfterCrossing) - MeanU;
const float3 Weights = float3(BeforeWeight, OwnWeight, AfterWeight);
const float Svv = dot(Weights, DeltaV * DeltaV);
const float Svu = dot(Weights, DeltaV * DeltaU);
const float Slope = (Svu + SlopeWeight * Own.Slope) / (Svv + SlopeWeight);

Candidate.Fit.Slope = Slope;
Candidate.Fit.Intercept = OwnCrossing - 0.5 * Slope;
```

With equal weights and both neighbours accepted, $\hat b = (u_B - u_A)/2$:
on the static edge 0, and the sawtooth is gone. With a neighbour on one side
only, an edge end or a corner, the own fit is kept as it is. A one-sided
test cannot tell a slanted edge from a parallel one without using the own
slope, which is the very thing the filter distrusts, and edge ends are rare.
With neither, the own fit is kept too.

Only the slope is replaced, as the last two lines say; the own centre
crossing stays. Replacing the crossing as well would smooth genuine position
disagreement between rows, but a position change is a fraction of a pixel
that coverage painting turns into a few percent of thickness, and keeping
the crossing has a second use: the dense edge suppression below compares
crossings, so its comparison is the same with or without the filter.

### Why the Within-Pixel Slopes Get Almost No Weight

Suppose for a moment that the rows' errors were independent. The slope from
the two outer crossings has variance

$$
\mathrm{Var}(\hat b_{\text{rows}}) = \frac{\mathrm{SE}(u_A)^2 + \mathrm{SE}(u_B)^2}{4} = \frac{2 \times 0.075^2}{4} = 0.053^2 ,
$$

against $0.26^2$ for the target's own slope. Inverse-variance weighting,
the minimum-variance combination of independent estimates, gives the
within-pixel slope

$$
\frac{0.053^2}{0.26^2 + 0.053^2} = 4\% .
$$

Now drop the independence assumption, which the static edge violates: the
rows' errors are shared. A shared error in the crossings moves all three
points by the same amount, which the fit absorbs into the position and
leaves $\hat b$ untouched. So the slope through the crossings is *exact* on
the static edge, while the within-pixel slope is wrong by 0.37 in every row,
and averaging rows does not help it. The right weight of every within-pixel
slope, the target's and the neighbours', is therefore about zero whenever
both neighbours are accepted: 4% at most under independence, nothing under
the shared error. The rule "slope from the crossings, fall back to the own
slope as the gate closes" is as close to optimal as a $3 \times 3$ window
allows. One pass is enough, too: the iterations of my first filter existed to
reach farther along the edge for line averaging, which the shared error
makes pointless.

### Pooling Across the Axis Instead

The same idea can be applied one level down, to the records: pool the rows
above and below as records shifted by $\pm 1$ in $v$, and fit the pool.
Pooling three identical records shifted by $-1$, 0 and $+1$ leaves the
covariance unchanged, since each shift is a constant per copy, and raises
the spread across the axis from $\sigma_v^2 = \tfrac1{12}$ to
$\tfrac1{12} + \tfrac23 = \tfrac34$:

$$
b_{\text{pooled}} = \frac{\sigma_{uv}}{\sigma_v^2 + \tfrac23}
= b_{\text{within}} \times \frac{1/12}{3/4} = \frac{b_{\text{within}}}{9} .
$$

On the static edge the lean drops from 0.37 to 0.04 pixels, invisible, and
on a slanted edge the between-row differences supply the slope with
$\tfrac89$ of the weight, so it would fix the sawtooth too. Its drawbacks
decided against it: there is no gate against a parallel edge or a corner in
the next row short of fitting that row anyway; it changes the fit that the
axis selection and the band test depend on; and it triple-counts samples in
the standard errors. The fit-level version costs the same record loads,
three pooled fits per axis instead of one, keeps the selection untouched and
gates explicitly.

## What the Filter Changed

Down the sawtooth column of the static frame, the alpha of the horizontal
axis's edges before and after the filter:

| Display rows | Column 235, before | after | Column 236, before | after |
|---|---|---|---|---|
| even (398, 400, ...) | 0.85 | 0.64 | 0.21 | 0.35 |
| odd (399, 401, ...) | 0.48 | 0.62 | 0.58 | 0.37 |

Rendering column 117's slope went from 0.370 to 0.014 and its centre
crossing stayed at 0.930. In the final image, display columns 225 to 246
and rows 390 to 415 of the 50% frame, zoomed eight times, with the filter
off on the left and on on the right:

![Two side-by-side crops of the same purple block corner, each 22 by 26 display pixels zoomed eight times and separated by a magenta bar. On the left the dark vertical outline running down the block's edge is a staircase of alternating darker and lighter pixel pairs, a two-row sawtooth; on the right the same outline is a straight, evenly dark column](/assets/images/thin-outline/sawtooth-filter-off-on.png)

The change is hard to see at a glance and clear pixel by pixel, which is why
I first built the filter for creases only and then had it applied to
silhouettes too. On that 50% frame, 3,197 of 3,622 drawn crease pixels had
valid crease fits in both neighbour rows and were filtered. Their slope
changed by 0.11 pixels per pixel at the median, the shared error of the
static edges, and by more than 0.5 in 235 pixels, all with an own slope
standard error above 3: records with few distinct samples whose own slope
was noise, on edges whose neighbour crossings agreed, so the filter
straightened them. Silhouettes moved less: 4,443 of 4,570 drawn silhouette
pixels were filtered, with a median slope change of 0.07 (90th percentile
0.22) and only 8 above 0.5. Their records are the better converged ones,
since the foreground keeps them and the kept ratio decides the side, so
their own slopes were nearer the neighbours' to begin with.

The cost, at 1280×720 on an RTX 5080 over a static view, as the median of
600-frame profiles:

| Drawing mode | Filter off | Filter on |
|---|---|---|
| before the upscaler, 100% | 0.137 ms | 0.148 ms (+8%) |
| after the upscaler, 50% | 0.076 ms | 0.085 ms (+12%) |

Silhouettes cost more than creases: half of the drawn pixels are
silhouettes, and a silhouette neighbour row pools up to four records with
their kept ratios against a crease row's three without. Under motion the
neighbour rows' fits come from the same reprojected records as the pixel's
own, so nothing new moves; a disocclusion resets rows unevenly, which the
gate and the inverse-variance weights are there to cover.

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
spatial filter weighs by. Within the tolerance the two pixels have
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
instead, in the
[flicker suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %})
post. The narrow background gap between two foreground parts of the
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
These are the spatial filter's two neighbour fits, so the test starts from
them and loads one more pooled record only when exactly one neighbour
continues:

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
the spike filter of the flicker post were in: some surfaces with grooves
still flickered at certain distances, the presence ramp could not be tuned
for them without making other creases at the same distance dashed, and the
specks were the handle that was left. Sharing its loads with the filter
made it nearly free, which is the other reason it stayed.

## What Each Step Is For

The three steps run in this order, once per rendering pixel, on the edge
the axis selection chose and before the band's coverage is computed: the
filter first, since it keeps the centre crossing that the dense test
compares; the dense test before the isolated one, since it is cheaper and
decides more pixels.

| Step | What it fixes | What it does not |
|---|---|---|
| Spatial filter | The shared slope error: the sawtooth below 100% after the upscaler, and noisy slopes of records with few distinct samples | Position disagreement between rows (the crossing is kept); the flicker |
| Dense edge suppression | Tangles of lines where three or more edges crowd within two pixels | The flicker of dense edges; isolated pairs, including one-pixel geometry |
| Isolated edge suppression | Specks of one or two pixels | The flicker of lines; it removes 0.6% of the outline |

None of them fixed the flicker after the upscaler, and that was what each
was tried against first. What did is in the
[flicker suppression]({% post_url 2026-10-11-thin-outline-flicker-suppression %})
post: the presence, the normal-spike filter and the thin-foreground filter,
all of which act on the records or at sampling time rather than on the drawn
line. The
[drawing and performance]({% post_url 2026-10-11-thin-outline-drawing-and-performance %})
post has the pass these steps run in and what they cost under the tile
classification, and the
[core idea]({% post_url 2026-10-11-thin-outline-core-idea %}) post has the
standard errors every one of them weighs or compares by.
