# SKILL: `extract_vecplot`

## Purpose

This skill teaches an AI system how to extract **point estimates, plotted uncertainty, derived confidence intervals, and treatment contrasts from vectorized scientific figures** when the numerical values are not printed for every plotted time point.

The primary use case is a longitudinal research figure in a PDF in which:

- each treatment group has a trajectory over time;
- each visit has a marker representing a mean, least-squares mean, model prediction, proportion, or other point estimate;
- vertical error bars represent some uncertainty quantity;
- one or more values are reported numerically elsewhere in the article and can be used as calibration anchors;
- intermediate values are only shown graphically; and
- the figure is vectorized, so marker centers, trajectory vertices, error-bar caps, axis ticks, and other objects have recoverable coordinates.

The central idea is to establish a numerical mapping between **figure coordinates** and **real-world values**, then apply that mapping to unlabelled graphical objects.

This is a geometry-based calibration procedure, not visual eyeballing.

A critical extension beyond a simple confidence-interval digitization workflow is that **error bars must first be classified correctly**. They may represent:

- standard errors;
- 95% confidence intervals;
- another confidence level;
- standard deviations;
- credible intervals;
- interquartile ranges;
- ranges; or
- an unspecified quantity.

The statistical interpretation determines what can be reconstructed from the geometry.

---

# 1. Core principle

A vectorized scientific plot contains graphical objects with exact or near-exact coordinates in the PDF/SVG coordinate system. On a linear y-axis, the real-world value and graphical vertical coordinate are related by an affine transformation:

\[
V = a + b y,
\]

where:

- \(V\) is the real-world outcome value;
- \(y\) is the transformed page-level vertical coordinate;
- \(a\) is the intercept; and
- \(b\) is the vertical scale factor.

Depending on the PDF/SVG coordinate convention, larger graphical \(y\)-coordinates can correspond to smaller numerical values. Therefore, never assume the sign of \(b\); estimate it from anchors.

For two numerical-coordinate anchors \((y_1,V_1)\) and \((y_2,V_2)\),

\[
b = \frac{V_2-V_1}{y_2-y_1},
\qquad
 a = V_1-b y_1.
\]

Then any unknown coordinate \(y_*\) maps to

\[
\widehat V_* = a+b y_*.
\]

The same idea applies to the x-axis when follow-up time is encoded continuously:

\[
T = c+d x.
\]

When visits are categorical rather than continuously spaced, map each x-coordinate to the printed visit label rather than interpolating time from distance.

---

# 2. The statistical interpretation comes before the geometry

Before extracting any coordinate, determine what the panel represents statistically.

Record all of the following:

- endpoint definition;
- outcome scale;
- whether the plotted quantity is an absolute value, change from baseline, percentage change, ratio, log-ratio, probability, or another measure;
- treatment groups;
- follow-up visits;
- analysis population;
- estimand;
- model type;
- whether the markers represent raw means, least-squares means, marginal means, medians, model predictions, or another estimand;
- how intercurrent events and missing outcomes were handled;
- what the error bars represent;
- the confidence level, if applicable; and
- whether the panel also contains values from another estimand or model.

For clinical-trial figures, the same panel can contain a longitudinal efficacy-estimand trajectory and a separate final treatment-regimen-estimand display after an x-axis break. These are not interchangeable.

## 2.1 Error-bar classification is mandatory

Read the caption, figure legend, methods, table footnotes, and supplementary material. Classify the plotted bars as one of:

```text
standard_error
confidence_interval_95
confidence_interval_other
standard_deviation
credible_interval
interquartile_range
range
unknown
```

Do not call an error bar a 95% CI merely because it looks like one.

## 2.2 Consequences of the classification

- **SE bars:** the plotted cap values are usually \(\widehat\mu\pm SE\). A 95% CI can be derived as \(\widehat\mu\pm 1.96SE\) only under an appropriate large-sample normal approximation.
- **95% CI bars:** transform the cap coordinates directly to obtain the CI. An SE can be approximated from the width only if the CI construction permits it.
- **SD bars:** do not reinterpret them as SEs or CIs. For a simple raw mean with known independent sample size, \(SE=SD/\sqrt n\) may be possible, but this is generally not valid for model-adjusted means.
- **Credible intervals:** report them as credible intervals. Do not convert mechanically to frequentist SEs.
- **IQR or range:** report the extracted interval as such.
- **Unknown bars:** extract the graphical bounds but label their statistical meaning as unknown.

---

# 3. Preferred source hierarchy

## 3.1 Numerical information

Retrieve legitimate numerical anchors from the article before inferring unlabelled values.

Preferred sources, in order:

1. Main-text numerical results.
2. Primary or supplementary tables.
3. Exact labels printed in the figure.
4. Figure caption.
5. Supplementary text.
6. Trial protocol or statistical analysis plan, when necessary to interpret the plotted quantity.

For each anchor, verify that it matches the plotted series with respect to:

- endpoint;
- estimand;
- model;
- population;
- treatment group;
- time point;
- analysis scale; and
- uncertainty type.

Do not calibrate an efficacy-estimand MMRM trajectory with a treatment-regimen-estimand ANCOVA value merely because both occur at week 72.

## 3.2 Graphical information

Prefer, in order:

1. Original vector PDF.
2. SVG exported from the original PDF.
3. Another lossless vector format.
4. High-resolution raster only when vector extraction fails.

If vector geometry exists, do not begin with screenshot pixels.

## 3.3 Independence rule

For an independent repeated extraction, do not use a previous AI's inferred intermediate values or coordinates as anchors. Use only:

- the original article;
- the original figure;
- author-reported numerical values; and
- the extraction methodology.

---

# 4. Coordinate terminology

For group \(g\) at visit \(t\), define:

- \(x_{g,t}\): horizontal coordinate of the visit marker;
- \(y_{m,g,t}\): vertical coordinate of the marker center or trajectory vertex;
- \(y_{\mathrm{top},g,t}\): coordinate of the geometrically top error-bar cap centerline;
- \(y_{\mathrm{bottom},g,t}\): coordinate of the geometrically bottom error-bar cap centerline;
- \(\mu_{g,t}\): real-world point estimate;
- \(E_{\mathrm{top},g,t}\): numerical value obtained by transforming the top cap;
- \(E_{\mathrm{bottom},g,t}\): numerical value obtained by transforming the bottom cap.

After applying \(V(y)=a+by\), define the numerical plotted bounds robustly as

\[
L^{\mathrm{plot}}_{g,t}
=
\min\{E_{\mathrm{top},g,t},E_{\mathrm{bottom},g,t}\},
\]

\[
U^{\mathrm{plot}}_{g,t}
=
\max\{E_{\mathrm{top},g,t},E_{\mathrm{bottom},g,t}\}.
\]

This avoids confusion when page coordinates increase downward.

For a reference visit \(r\), the article might provide:

\[
\mu_{g,r},\qquad SE_{g,r},
\]

or

\[
\mu_{g,r},\qquad L_{g,r},\qquad U_{g,r}.
\]

The figure supplies the corresponding vector coordinates.

---

# 5. Essential distinction: the estimate is at the marker center

The numerical point estimate is represented by the **logical center of the marker**, not an edge or vertex of the visible shape.

Examples:

- circle: use \((c_x,c_y)\);
- square: use its geometric center;
- diamond: use the midpoint of opposing vertices or the trajectory-line vertex through its center;
- triangle: use the plotting origin or trajectory vertex, not the top tip;
- open marker: use the geometric center of the stroke outline;
- horizontal error-bar cap: use the cap's centerline coordinate;
- vertical error-bar stem: use endpoints only as a cross-check;
- trajectory path: at a visit, its vertex should coincide with the marker center.

## 5.1 The marker-edge fallacy

For a diamond, the top vertex is displaced from the true point estimate by half the marker height. Using that vertex systematically biases every extracted mean. The same problem applies to the top edge of a square and the border of a circle.

If a diamond has top and bottom vertices \(y_T\) and \(y_B\), its center is

\[
y_m=\frac{y_T+y_B}{2}.
\]

When the trajectory path passes through the marker, its visit vertex is often the cleanest center coordinate.

## 5.2 Stroke width

A visible line has thickness. Use the vector centerline, not the rendered upper or lower edge of the stroke.

---

# 6. End-to-end workflow

The recommended workflow is:

1. Read the caption and methods.
2. Identify the endpoint, estimand, model, and error-bar type.
3. Identify the target panel.
4. Identify all treatment series by color, marker shape, and line style.
5. Identify all visit locations.
6. Retrieve independent numerical anchors.
7. Confirm that the source is vectorized.
8. Extract and transform page-level coordinates.
9. Establish the x-axis visit mapping.
10. Establish the y-axis numerical calibration.
11. Recover marker-center values.
12. Recover error-bar cap values.
13. Interpret the recovered bars according to their declared type.
14. Derive SEs or 95% CIs only when statistically justified.
15. Compute treatment contrasts on the correct scale.
16. Address covariance between arm-specific estimates.
17. Validate against every available reported value.
18. Report source type, extraction method, assumptions, and precision.

---

# 7. Obtaining vector geometry

A common workflow is:

```text
PDF page -> SVG or PDF drawing objects -> transformed page coordinates
```

Possible tools include:

- `pdftocairo -svg`;
- `mutool draw -F svg`;
- a PDF graphics library such as PyMuPDF;
- direct parsing of SVG/XML paths; or
- another library that exposes PDF drawing primitives.

The exact software is less important than recovering:

- path definitions;
- line and fill colors;
- stroke widths;
- marker shapes;
- trajectory vertices;
- error-bar stems and caps;
- clipping paths;
- transformation matrices; and
- reused symbols or `<use>` elements.

Do not rely only on extracted text. Text extraction rarely exposes the underlying plot geometry.

---

# 8. Resolve all coordinate transformations

SVG and PDF elements are often defined in local coordinates and transformed into page space.

For an affine transformation,

\[
\begin{pmatrix}
 x'\\
 y'
\end{pmatrix}
=
\begin{pmatrix}
 a & c\\
 b & d
\end{pmatrix}
\begin{pmatrix}
 x\\
 y
\end{pmatrix}
+
\begin{pmatrix}
 e\\
 f
\end{pmatrix}.
\]

Apply the complete chain of parent and child transformations before comparing objects.

Do not compare raw local coordinates from elements with different transforms.

Pay attention to:

- nested `<g transform=...>` groups;
- page rotation;
- y-axis reversal;
- clipping regions;
- reused marker symbols;
- local symbol origins;
- translated `<use>` objects; and
- separate coordinate systems for different panels.

All coordinates used in calibration must be in one common page coordinate system.

---

# 9. Identify the target panel and series

A journal page can contain multiple panels, legends, labels, and repeated colors. First isolate the target panel's bounding box or clipping region.

For each treatment group, record:

```text
group_name
stroke_color
fill_color
marker_shape
line_style
error_bar_style
legend_location
```

Use color together with marker and line style. Do not rely on color alone.

A practical series map might be:

```text
Treatment A -> cyan -> downward triangle -> solid line
Treatment B -> maroon -> diamond -> solid line
Placebo     -> gray -> square -> solid line
```

Open markers may have white fill and colored stroke. Error bars may use a lighter opacity than the trajectory. Identify objects structurally and spatially, not solely through exact RGB equality.

---

# 10. Identify visit coordinates

Preferred procedure:

1. extract x-axis tick locations;
2. pair them with printed visit labels;
3. find the marker or trajectory vertices at the same x-locations;
4. verify the visit order;
5. distinguish true visit points from post-break symbols.

If the axis represents actual numeric time and spacing is proportional, fit

\[
T=c+dx.
\]

If visits are displayed as equally spaced categories despite unequal week gaps, do not infer time from distance. Match by tick position and order.

## 10.1 Broken x-axis

A symbol displayed after a break may represent:

- a different estimand;
- an alternative analysis;
- a separate final-visit summary; or
- another population.

Do not treat it as the next point in the longitudinal sequence.

---

# 11. Locate the trajectory path

A longitudinal trajectory is often encoded as a polyline or path:

```text
M x0 y0
L x1 y1
L x2 y2
...
L xK yK
```

After resolving transformations, the ordered vertices can provide marker-center coordinates at all visits.

Confirm that:

- path x-coordinates align with visit ticks;
- path vertices intersect marker centers;
- the path color matches the intended group; and
- the sequence is not a confidence-band boundary or another graphical object.

The trajectory path is often more precise than reconstructing centers from marker outlines one by one.

---

# 12. Locate error-bar geometry

At each visit, an error bar can comprise:

- a top horizontal cap;
- an upper vertical stem;
- a lower vertical stem;
- a bottom horizontal cap; and
- a central marker.

Extract the centerline y-coordinates of the top and bottom caps.

Use the following checks:

- both endpoints of a horizontal cap should share the same y-coordinate;
- the cap should be centered approximately at the visit x-coordinate;
- the stem should meet the cap centerline;
- the marker should lie between the caps; and
- cap color and line width should be consistent with the series.

If caps are absent and only a vertical stem is drawn, use the stem endpoints, recognizing that line-cap style can create a small rendering offset.

---

# 13. Y-axis calibration strategies

Use more than one strategy whenever possible.

## 13.1 Strategy A: axis-tick calibration

If y-axis tick centerlines and printed values are identifiable, fit

\[
V=a+by.
\]

With two ticks, solve exactly. With three or more, use least squares and inspect residuals.

Axis ticks are often the best global calibration because their printed values are conceptually exact.

## 13.2 Strategy B: baseline plus known reference marker

For change-from-baseline plots, baseline often satisfies \(V_0=0\). If the same series has a known reference value \(V_r\) at coordinate \(y_r\), and baseline is at \(y_0\), then

\[
V_t
=
V_0+
(V_r-V_0)
\frac{y_t-y_0}{y_r-y_0}.
\]

When \(V_0=0\),

\[
V_t
=
V_r
\frac{y_t-y_0}{y_r-y_0}.
\]

This is the canonical relative-distance calculation.

Use the baseline marker center or the zero-axis/dashed-line centerline, not the top edge of a marker.

## 13.3 Strategy C: known reference mean and uncertainty cap

If axis ticks are unavailable but a reference mean and a known uncertainty bound are reported, the vertical scale magnitude can be inferred from their graphical separation.

For example, with a known upper numerical bound \(U_r\),

\[
|b|
=
\frac{|U_r-\mu_r|}
{|y_{\mathrm{top},r}-y_{m,r}|},
\]

provided the top cap corresponds to \(U_r\). Use the sign implied by axis direction.

Repeat with the lower bound and compare.

## 13.4 Strategy D: multiple-anchor linear system

Suppose \(K\) anchors are available:

\[
V_j=a+b y_j,\qquad j=1,\ldots,K.
\]

Write

\[
\mathbf V=
\begin{pmatrix}
V_1\\
\vdots\\
V_K
\end{pmatrix},
\qquad
\mathbf X=
\begin{pmatrix}
1&y_1\\
\vdots&\vdots\\
1&y_K
\end{pmatrix},
\qquad
\boldsymbol\beta=
\begin{pmatrix}
a\\b
\end{pmatrix}.
\]

Then

\[
\mathbf V=\mathbf X\boldsymbol\beta.
\]

For exactly two independent anchors, solve the system exactly. For more than two,

\[
\widehat{\boldsymbol\beta}
=
(\mathbf X^T\mathbf W\mathbf X)^{-1}
\mathbf X^T\mathbf W\mathbf V,
\]

where \(\mathbf W\) can give greater weight to exact axis ticks and lower weight to rounded article values.

Inspect residuals:

\[
r_j=V_j-(\widehat a+\widehat b y_j).
\]

Large residuals suggest a wrong object, wrong estimand, transformation error, nonlinear axis, or article rounding beyond expectation.

## 13.5 Strategy E: transformed axes

For a log axis, the affine relation applies on the log scale:

\[
\log V=a+by.
\]

Calibrate in transformed space, then back-transform.

Do not apply a linear raw-value mapping to a nonlinear axis.

---

# 14. Recovering point estimates

After calibrating \(V(y)=a+by\), transform each marker-center coordinate:

\[
\widehat\mu_{g,t}=a+b y_{m,g,t}.
\]

Also compute a reference-marker result when possible:

\[
\widehat\mu_{g,t}^{\mathrm{ref}}
=
\mu_{g,0}
+
(\mu_{g,r}-\mu_{g,0})
\frac{y_{m,g,t}-y_{m,g,0}}
{y_{m,g,r}-y_{m,g,0}}.
\]

The axis-derived and reference-derived estimates should agree within tolerances justified by printed rounding.

Do not force agreement by moving coordinates manually.

---

# 15. Recovering the plotted bounds

Transform both cap coordinates:

\[
E_{\mathrm{top},g,t}=a+b y_{\mathrm{top},g,t},
\]

\[
E_{\mathrm{bottom},g,t}=a+b y_{\mathrm{bottom},g,t}.
\]

Then sort them numerically:

\[
L^{\mathrm{plot}}_{g,t}
=
\min(E_{\mathrm{top},g,t},E_{\mathrm{bottom},g,t}),
\]

\[
U^{\mathrm{plot}}_{g,t}
=
\max(E_{\mathrm{top},g,t},E_{\mathrm{bottom},g,t}).
\]

Define the upper and lower graphical half-widths:

\[
h_{+,g,t}=U^{\mathrm{plot}}_{g,t}-\widehat\mu_{g,t},
\]

\[
h_{-,g,t}=\widehat\mu_{g,t}-L^{\mathrm{plot}}_{g,t}.
\]

Slight asymmetry can arise from rounding or vector-coordinate precision. Preserve the direct transformed values and report a symmetric summary only when justified.

---

# 16. Interpreting standard-error bars

If the caption states that error bars indicate standard error, the plotted interval is generally

\[
\widehat\mu_{g,t}\pm SE_{g,t}.
\]

Estimate the SE from the geometry as

\[
\widehat{SE}_{g,t}
=
\frac{h_{+,g,t}+h_{-,g,t}}{2}.
\]

If the bars are highly symmetric, either half-width alone should agree.

The extracted cap interval must be described as **mean ±1 SE**, not as a 95% CI.

Under an appropriate normal approximation, derive an approximate 95% CI:

\[
CI_{95\%,g,t}
=
\widehat\mu_{g,t}
\pm 1.96\widehat{SE}_{g,t}.
\]

If the model uses a t critical value with known degrees of freedom, use

\[
\widehat\mu_{g,t}
\pm t_{0.975,\nu}\widehat{SE}_{g,t}.
\]

Label a normal-based interval as approximate when the exact critical value is not known.

---

# 17. Interpreting confidence-interval bars

If the bars are a two-sided 95% CI, the direct transformed bounds are the target interval:

\[
CI_{95\%,g,t}
=
\left[
L^{\mathrm{plot}}_{g,t},
U^{\mathrm{plot}}_{g,t}
\right].
\]

If the CI is approximately symmetric and normal-based,

\[
\widehat{SE}_{g,t}
\approx
\frac{U^{\mathrm{plot}}_{g,t}-L^{\mathrm{plot}}_{g,t}}
{2\times1.96}.
\]

Equivalently,

\[
\widehat{SE}_{g,t}
\approx
\frac{h_{+,g,t}+h_{-,g,t}}{2\times1.96}.
\]

Do not use 1.96 when the article specifies:

- a different confidence level;
- a t critical value;
- bootstrap percentile intervals;
- profile-likelihood intervals;
- transformed-scale intervals;
- Bayesian intervals; or
- another construction.

---

# 18. Interpreting SD, IQR, range, or other bars

## 18.1 Standard deviation

If bars represent SD:

\[
L^{\mathrm{plot}}=\widehat\mu-SD,
\qquad
U^{\mathrm{plot}}=\widehat\mu+SD.
\]

Do not derive a CI unless the relevant sampling design and sample size justify an SE calculation.

For a simple unadjusted mean based on \(n\) independent observations,

\[
SE=\frac{SD}{\sqrt n}.
\]

This formula is generally inappropriate for MMRM least-squares means or other adjusted model estimates.

## 18.2 IQR, range, and quantile intervals

Transform and report the plotted bounds using their actual interpretation. Do not relabel them as uncertainty about the mean.

## 18.3 Unknown bar type

Report:

```text
point estimate
lower plotted bound
upper plotted bound
error-bar meaning: not identified
```

Do not infer an SE or CI.

---

# 19. Relative-distance reconstruction without an explicit axis equation

Sometimes the most convenient demonstration uses relative distances from a known baseline and a known final estimate.

Suppose:

- baseline value is \(0\) at \(y_0\);
- a reference mean is \(\mu_r\) at \(y_r\); and
- the target mean is at \(y_t\).

Then

\[
\widehat\mu_t
=
\mu_r
\frac{y_t-y_0}{y_r-y_0}.
\]

Equivalently, compute the per-coordinate conversion factor

\[
s
=
\frac{\mu_r}{y_r-y_0},
\]

then

\[
\widehat\mu_t=s(y_t-y_0).
\]

This method and the affine-axis method are mathematically equivalent when the baseline is zero.

For error bars, use the same calibrated scale on cap centerlines. Do not calibrate means from marker centers but uncertainty from marker edges.

---

# 20. Computing a treatment effect from reconstructed arm means

For a continuous endpoint on an identity scale, define the active-versus-control contrast at visit \(t\) as

\[
\Delta_t=\mu_{A,t}-\mu_{C,t}.
\]

The reconstructed estimate is

\[
\widehat\Delta_t
=
\widehat\mu_{A,t}-\widehat\mu_{C,t}.
\]

For signed percent change in bodyweight, a more negative value indicates greater reduction. It is often useful to report both:

- signed effect: active minus control; and
- additional weight loss: the same magnitude with a positive verbal interpretation.

## 20.1 Do not subtract CI endpoints

The ordinary CI for a difference is not obtained by subtracting one arm's upper limit from the other arm's lower limit.

The contrast variance is

\[
\operatorname{Var}(\widehat\Delta_t)
=
SE_{A,t}^2
+
SE_{C,t}^2
-
2\operatorname{Cov}
(\widehat\mu_{A,t},\widehat\mu_{C,t}).
\]

Therefore,

\[
SE_{\Delta,t}
=
\sqrt{
SE_{A,t}^2
+
SE_{C,t}^2
-
2\operatorname{Cov}
(\widehat\mu_{A,t},\widehat\mu_{C,t})
}.
\]

## 20.2 When zero covariance is exact or approximate

For two completely independent unadjusted sample means from disjoint groups, zero covariance can be exact under the assumed sampling model.

For least-squares means estimated from the same regression or MMRM, covariance is generally not guaranteed to be zero because the means share nuisance-parameter and covariance estimates.

If the model covariance matrix is unavailable, the transparent approximation is

\[
\operatorname{Cov}
(\widehat\mu_{A,t},\widehat\mu_{C,t})
\approx0,
\]

so

\[
SE_{\Delta,t}
\approx
\sqrt{SE_{A,t}^2+SE_{C,t}^2}.
\]

Then an approximate 95% CI is

\[
\widehat\Delta_t
\pm1.96SE_{\Delta,t}.
\]

State the covariance assumption explicitly.

## 20.3 Back-solving covariance from a reported reference contrast

If the paper reports a direct active-versus-control contrast at a reference visit \(r\), use it to assess the covariance approximation.

First infer the reference contrast SE from its CI, when appropriate:

\[
SE_{\Delta,r}
\approx
\frac{U_{\Delta,r}-L_{\Delta,r}}
{2\times1.96}.
\]

Then

\[
\widehat{\operatorname{Cov}}_r
=
\frac{
SE_{A,r}^2+SE_{C,r}^2-SE_{\Delta,r}^2
}{2}.
\]

The implied correlation is

\[
\widehat\rho_r
=
\frac{
\widehat{\operatorname{Cov}}_r
}{SE_{A,r}SE_{C,r}}.
\]

This provides a powerful quality-control check.

If \(\widehat\rho_r\) is essentially zero, the zero-covariance approximation at nearby visits may be defensible. If it is not, report the limitation or use the reference correlation as a sensitivity assumption:

\[
SE_{\Delta,t}(\rho_r)
=
\sqrt{
SE_{A,t}^2+SE_{C,t}^2
-2\rho_r SE_{A,t}SE_{C,t}
}.
\]

Transporting a correlation across visits is an approximation and must be labeled as such.

## 20.4 Prefer a directly reported contrast

If the article directly reports the target visit's treatment contrast and CI, use that result rather than reconstructing it from arm means.

---

# 21. Contrasts on non-identity scales

Subtracting arm means is appropriate for a mean difference on an identity scale. Other estimands require the correct scale.

Examples:

- log-risk: subtract log risks, then exponentiate for a risk ratio;
- log-odds: subtract logits, then exponentiate for an odds ratio;
- log-rate: subtract log rates, then exponentiate for a rate ratio;
- log-hazard: use the model's log-hazard contrast;
- percentage response: determine whether the target is risk difference, risk ratio, or odds ratio.

Do not infer a model-based odds ratio merely by subtracting plotted probabilities.

If the plotted arm values are already back-transformed model means, the exact contrast SE may require the delta method or the model's covariance matrix.

---

# 22. Multiple anchors and rounding-aware calibration

Published anchors are often rounded to one decimal place, whereas figure coordinates were generated from unrounded values.

A printed value \(v\) rounded to one decimal represents an underlying value approximately in

\[
[v-0.05,v+0.05).
\]

Practical recommendations:

1. use exact axis ticks as primary scale anchors when available;
2. use rounded article results as validation anchors;
3. fit multiple anchors by weighted least squares;
4. preserve full vector-coordinate precision internally;
5. do not overreact to discrepancies of a few hundredths; and
6. report final precision consistent with source rounding.

If several rounded anchors all support the same affine mapping, the geometry is likely correctly identified.

---

# 23. Redundant calibration is a feature

Use independent checks whenever possible.

## Check 1: y-axis ticks

Estimate \(a\) and \(b\) from printed ticks.

## Check 2: zero baseline

Confirm that baseline markers or the baseline dashed line transform to zero.

## Check 3: known final means

Transform final marker centers and compare with reported values.

## Check 4: known final uncertainty

Transform final caps and compare with reported SEs or CIs.

## Check 5: other treatment groups

Apply the same global mapping to every group.

## Check 6: bar symmetry

If bars represent \(\pm SE\), compare the two cap-to-center distances.

## Check 7: direct treatment contrast

Reconstruct the final active-minus-control difference and its approximate SE; compare with the reported direct contrast.

## Check 8: cross-method agreement

Compare axis-tick, baseline-reference, and uncertainty-cap calibrations.

Do not report high numerical precision unless the checks are mutually consistent.

---

# 24. Object-identification pitfalls

## 24.1 Marker center versus marker tip

Never use the top vertex of a diamond or triangle as the mean coordinate.

## 24.2 Error-bar cap versus marker edge

A marker border can resemble a horizontal cap. Confirm that the candidate cap extends beyond the marker and is connected to a stem.

## 24.3 Open markers

Search both stroke and fill attributes.

## 24.4 Reused symbols

SVG may define one marker in `<defs>` and place it through many `<use>` elements. The translation of each use object identifies its center.

## 24.5 Error bars behind markers

The central stem can be obscured. The top and bottom caps are usually still extractable.

## 24.6 Similar colors

Use shape, x-position, and trajectory connectivity in addition to RGB.

## 24.7 PDF coordinate direction

Infer direction from tick values rather than assuming it.

## 24.8 Multiple panels

The same x and y values in different panels are not comparable until transformed into common page coordinates and assigned to the correct panel.

## 24.9 Broken axes and separate estimands

Do not bridge a graphical axis break with a continuous calibration.

## 24.10 Labels mistaken for data objects

Legend markers, figure annotations, and endpoint labels can have the same color and shape as the actual series. Restrict extraction to the plot's clipping region.

---

# 25. When the axis itself is enough

If multiple y-axis ticks can be recovered accurately, the entire panel can be calibrated without using a reported final value.

This is often preferable because:

- tick values are exact by construction;
- the same scale applies globally;
- arm-specific anchors may be rounded; and
- error bars can be transformed directly regardless of their statistical type.

Reported final values remain essential for validation and interpretation.

---

# 26. When a same-series reference anchor is preferable

A baseline-plus-final reference calibration is useful when:

- axis ticks are embedded as text but their tick-line coordinates are difficult to identify;
- the panel is cropped;
- the axis is visually compressed;
- the final value is reported exactly; or
- an independent same-series check is desired.

Because it uses the same trajectory, it is robust to some panel-level translation ambiguities. It is still vulnerable to using the wrong marker point or wrong estimand.

---

# 27. Raster fallback

If no vector source exists, the same affine method can be applied to high-resolution raster pixels.

Recommended steps:

1. render at high DPI;
2. avoid lossy screenshots;
3. identify tick centerlines;
4. identify marker centers rather than edges;
5. use cap centerlines;
6. repeat measurements at multiple zoom levels;
7. quantify pixel-level uncertainty; and
8. report lower confidence in the extracted precision.

A one-pixel error maps to

\[
|b|
\]

outcome units. Include that resolution in the uncertainty assessment.

Do not describe raster digitization as exact vector extraction.

---

# 28. Suggested automated algorithm

```text
INPUT:
    original article PDF
    vector figure or vector article page
    supplementary appendix or tables
    target panel and visits

A. INTERPRET THE PANEL
    identify endpoint
    identify estimand
    identify analysis model
    identify analysis population
    identify error-bar type
    identify treatment groups and visits

B. RETRIEVE LEGITIMATE ANCHORS
    retrieve exact axis tick labels
    retrieve reported reference means
    retrieve reported reference SEs or CIs
    verify endpoint/model/estimand match

C. EXTRACT VECTOR OBJECTS
    isolate target panel
    parse paths, strokes, fills, transforms, and clip paths
    resolve all objects into page coordinates

D. IDENTIFY SERIES
    map group to color + marker + line style
    locate trajectory paths
    locate marker centers
    locate error-bar caps and stems

E. MAP VISITS
    extract x-axis tick coordinates
    pair x-coordinates with visit labels
    exclude post-break symbols

F. CALIBRATE Y
    fit affine mapping from axis ticks
    fit same-series baseline/reference mapping
    optionally solve a multi-anchor weighted system
    compare mappings

G. EXTRACT ARM RESULTS
    mean = transform(marker center)
    top_bound = transform(top cap)
    bottom_bound = transform(bottom cap)
    sort numerical bounds

H. INTERPRET UNCERTAINTY
    if bars are SE:
        SE = average cap-to-mean half-width
        derived 95% CI = mean +/- critical_value * SE
    if bars are 95% CI:
        use transformed bounds directly
        derive SE only if justified
    otherwise:
        preserve declared interval type

I. COMPUTE TREATMENT CONTRASTS
    contrast = active mean - control mean
    use exact model contrast if reported
    otherwise calculate contrast SE using covariance formula
    state or assess covariance assumption

J. QUALITY CONTROL
    validate known reference means
    validate known reference SEs/CIs
    validate baseline
    validate direct reference contrast
    inspect residuals and object assignments

OUTPUT:
    group-by-visit table
    treatment-contrast table
    extraction notes
    calibration diagnostics
    assumptions and limitations
```

---

# 29. Recommended output schema

For arm-specific results:

```text
treatment
week
estimand
model
mean
error_bar_type
plotted_lower
plotted_upper
extracted_se
derived_lower_95_ci
derived_upper_95_ci
source_type
extraction_method
```

For contrasts:

```text
week
comparison
contrast_scale
estimate
se
lower_95_ci
upper_95_ci
covariance_method
source_type
```

Suggested `source_type` values:

```text
reported_textually
reported_in_table
reported_on_figure
vector_inferred
raster_inferred
```

Suggested `extraction_method` values:

```text
direct_report
axis_tick_calibration
baseline_reference_calibration
reference_uncertainty_calibration
multi_anchor_affine_fit
```

Suggested `covariance_method` values:

```text
model_reported
back_solved_from_reference_contrast
assumed_zero
assumed_reference_correlation
unknown
```

---

# 30. Recommended table formats

## 30.1 Arm-specific reconstruction

```markdown
| Week | Group | Mean | Extracted SE | Plotted interval, mean ±1 SE | Derived approximate 95% CI |
|---:|---|---:|---:|---:|---:|
| ... | ... | ... | ... | ... | ... |
```

When the bars are CIs rather than SEs, replace the plotted-interval column accordingly.

## 30.2 Treatment contrast

```markdown
| Week | Estimated treatment effect | Approx. SE | Approximate 95% CI |
|---:|---:|---:|---:|
| ... | ... | ... | ... |
```

Include a note defining the sign and covariance assumption.

---

# 31. Quality-control tolerances

Tolerances depend on source precision, but useful checks include:

- a reconstructed known mean should agree within expected publication rounding;
- reconstructed reference SEs should agree with printed SEs within rounding;
- reconstructed reference CIs should agree with printed CIs within rounding;
- affine-fit residuals for axis ticks should be near numerical zero;
- top and bottom SE-bar half-widths should be similar;
- all group markers at a visit should share the same x-coordinate unless intentionally dodged;
- no numerical lower bound should exceed the point estimate;
- no numerical upper bound should be below the point estimate;
- a known direct contrast should be reproduced approximately; and
- the implied contrast correlation should be plausible.

If one group fails while all others pass, suspect object misclassification rather than global scale failure.

---

# 32. Precision and uncertainty reporting

Vector coordinates may be available to many decimal places, but the underlying statistical values may be printed only to one decimal.

Recommended practice:

- retain full coordinates internally;
- retain two or three decimals in technical extraction calculations;
- report one or two decimals according to the source's precision;
- distinguish geometric precision from statistical certainty;
- label derived CIs as approximate when a normal critical value is assumed; and
- label treatment-effect SEs as approximate when covariance is not known exactly.

A high-precision coordinate is not proof that the original model estimate is known to the same decimal precision.

---

# 33. When not to use the method without modification

Do not apply the basic linear workflow unmodified when:

- the y-axis is logarithmic or nonlinear;
- the figure uses a transformed scale that is not recognized;
- the panel is a smoothed curve with no visit-specific marker;
- the source is a low-resolution raster with overlapping objects;
- the requested time lies between plotted visits and interpolation is not scientifically justified;
- markers are intentionally jittered or dodged and visit identity is uncertain;
- the figure was manually redrawn rather than generated from analysis output;
- the numerical anchor belongs to another estimand or population;
- uncertainty type cannot be identified; or
- the desired treatment contrast is on a non-identity scale and the necessary model covariance is unavailable.

In such cases, adapt the method or report that only an approximate graphical readout is possible.

---

# 34. Independence and anti-contamination rules

To support reproducible independent extractions:

1. Do not embed previous AI-extracted target coordinates in the skill.
2. Do not embed previous AI-inferred intermediate values.
3. Do not tell a new system what answer it should recover.
4. Provide the original article, figure, and legitimate reported anchors.
5. Require independent series identification.
6. Require independent coordinate extraction.
7. Require independent calibration.
8. Require validation against reported anchors.
9. Record software and transformation steps when reproducibility matters.

The goal is methodological reproducibility, not answer memorization.

---

# 35. Recommended result-reporting language

For vector-inferred arm values:

> Intermediate visit estimates were reconstructed from the vectorized published figure. The y-axis was calibrated using exact axis coordinates and independently reported reference values. Marker centers were used for point estimates, and error-bar cap centerlines were used for the plotted uncertainty bounds. These values are high-precision figure-derived estimates rather than originally tabulated numerical results.

For SE bars:

> The figure caption states that the error bars represent standard errors. The plotted bounds were therefore interpreted as mean ±1 SE. Approximate 95% confidence intervals were derived as mean ±1.96 SE.

For an approximate treatment contrast:

> The treatment effect was computed as the difference between the reconstructed arm-specific means. Its standard error was approximated from the two arm-specific SEs under the stated covariance assumption. The exact model-based contrast variance would require the original fitted-model covariance matrix.

---

# 36. Final checklist

Before returning results, verify all of the following:

- [ ] I identified the endpoint and outcome scale.
- [ ] I identified the estimand and analysis population.
- [ ] I identified the analysis model.
- [ ] I verified what the error bars represent.
- [ ] I did not automatically call SE bars confidence intervals.
- [ ] I retrieved legitimate numerical anchors.
- [ ] I confirmed that anchors match the plotted estimand and model.
- [ ] I verified that the source is vectorized, or clearly labeled a raster fallback.
- [ ] I isolated the correct panel.
- [ ] I identified each series by color, marker shape, and line style.
- [ ] I mapped x-coordinates to the correct visits.
- [ ] I resolved all transformations into page coordinates.
- [ ] I used marker centers, not marker tips or edges.
- [ ] I used error-bar cap centerlines.
- [ ] I calibrated the y-axis with an affine mapping.
- [ ] I performed at least one independent calibration check.
- [ ] I validated known reference means.
- [ ] I validated known reference SEs or CIs.
- [ ] I preserved the actual error-bar interpretation.
- [ ] I derived a 95% CI only when justified.
- [ ] I computed treatment contrasts on the correct scale.
- [ ] I did not subtract CI endpoints to form a contrast CI.
- [ ] I addressed covariance between arm-specific estimates.
- [ ] I clearly marked reconstructed versus directly reported values.
- [ ] I did not overstate numerical precision.

---

# 37. Conceptual summary

The fundamental calibration is

\[
\boxed{\text{known numerical value}}
\longleftrightarrow
\boxed{\text{known vector coordinate}}.
\]

Once the affine mapping is established,

\[
\boxed{\text{unknown vector coordinate}}
\longrightarrow
\boxed{\text{inferred numerical value}}.
\]

For point estimates, use the **marker center or trajectory vertex**.

For error bars, use the **cap centerlines** and preserve their declared statistical meaning.

For SE bars:

\[
\boxed{\text{plotted bar} = \widehat\mu\pm SE},
\]

and, when appropriate,

\[
\boxed{CI_{95\%}\approx\widehat\mu\pm1.96SE}.
\]

For a treatment difference on an identity scale:

\[
\boxed{\widehat\Delta=\widehat\mu_A-\widehat\mu_C},
\]

with

\[
\boxed{
SE_\Delta
=
\sqrt{SE_A^2+SE_C^2-2\operatorname{Cov}(\widehat\mu_A,\widehat\mu_C)}
}.
\]

The method is most powerful when the original source is a true vector PDF because it replaces subjective visual approximation with reproducible geometric reconstruction while remaining anchored to the statistical interpretation given by the article.
