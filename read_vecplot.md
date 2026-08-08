# SKILL: `read_vecplot`

## Purpose

This skill teaches an AI system how to extract **point estimates and interval estimates at specific follow-up times from a vectorized scientific figure** when the numerical values are not printed for every plotted time point.

The intended use case is a longitudinal line plot in a journal PDF in which:

- each treatment group has a plotted trajectory over time;
- each visit has a marker representing a model-based mean or other point estimate;
- vertical error bars represent a confidence interval;
- some time points, often a final follow-up visit, have exact numerical point estimates and confidence intervals reported elsewhere in the paper or tables;
- the intermediate time-point values are only visible graphically; and
- the PDF figure is vectorized, so the plotted marker centers, error-bar caps, and line coordinates can be recovered at high precision from the underlying vector geometry.

The core idea is to use **known real-world numerical values as calibration anchors** and infer unknown numerical values from the relative vector coordinates of plotted graphical objects.

This is a geometry-based calibration procedure, not a visual eyeballing procedure.

---

# 1. Core principle

A vectorized scientific plot contains graphical objects with exact or near-exact coordinates in the PDF/SVG coordinate system. If a plotted object's coordinate is associated with a real-world numerical value that is independently reported in the article, then the plot can be calibrated.

For a standard linear y-axis, the relationship between the graphical y-coordinate and the real-world outcome value is affine:

\[
V = a + b y,
\]

where:

- \(V\) is the real-world outcome value;
- \(y\) is the vectorized vertical coordinate;
- \(a\) is an intercept;
- \(b\) is the vertical scale factor.

Depending on the SVG/PDF coordinate convention, larger graphical y-coordinates may correspond to smaller numerical outcome values. Therefore, never assume the sign of \(b\); infer it from calibration points.

The same logic applies to the x-axis for identifying follow-up times:

\[
T = c + d x,
\]

where \(T\) is the real-world follow-up time and \(x\) is the vectorized horizontal coordinate.

When axis tick coordinates can be identified directly, the axis itself can be used for calibration. When exact numerical estimates are available for plotted markers or error bars, those graphical objects provide additional high-quality anchors.

---

# 2. Preferred source hierarchy

Use the following hierarchy of evidence.

## 2.1 Numerical anchor values

Obtain exact reference values from the article before inferring any unlabeled plotted values.

Preferred sources, in order:

1. Main-text numerical results.
2. Primary or supplementary tables.
3. Figure labels that explicitly print exact estimates.
4. Figure captions.
5. Supplementary text.

Do **not** use a prior AI's digitized result as an anchor when an independent extraction is desired.

For every anchor, verify that the numerical value corresponds to the **same estimand, analysis model, population, endpoint definition, treatment group, and time point** as the plotted series being calibrated.

For example, do not calibrate an efficacy-estimand MMRM trajectory using a treatment-regimen-estimand ANCOVA estimate merely because the treatment and nominal week are the same.

## 2.2 Graphical source

Prefer, in order:

1. Original vector PDF.
2. SVG exported from the PDF.
3. Another lossless vector format.
4. High-resolution raster only if vector extraction fails.

If the source is vectorized, work from vector geometry rather than screen pixels whenever possible.

---

# 3. Essential distinction: graphical center coordinates

The calibration coordinate must correspond to the **center location representing the numerical estimate**, not an arbitrary edge of the graphical shape.

Examples:

- filled circle: use the geometric center of the circle;
- filled diamond: use the geometric center of the diamond;
- triangle: use the center location used by the plotting routine, normally the trajectory vertex at that visit;
- open square: use the geometric center of the square;
- horizontal confidence-interval cap: use the vertical coordinate of the cap line, which is constant along the cap;
- vertical confidence-interval stem: use its upper and lower endpoints only as a secondary check;
- line trajectory: at a visit, the line vertex should coincide with the point-estimate marker center.

Do not use the upper or lower edge of a thick marker. Do not use the top or bottom edge created by stroke width. Use the object's logical centerline or center point in vector space.

---

# 4. Terminology used in this skill

For treatment group \(g\) and visit \(t\), define:

- \(y_{m,g,t}\): y-coordinate of the point-estimate marker center;
- \(y_{u,g,t}\): y-coordinate of the upper confidence-limit cap centerline;
- \(y_{l,g,t}\): y-coordinate of the lower confidence-limit cap centerline;
- \(\mu_{g,t}\): real-world point estimate;
- \(U_{g,t}\): real-world upper confidence limit;
- \(L_{g,t}\): real-world lower confidence limit.

For a known reference visit \(r\), the article supplies:

\[
\mu_{g,r}, \qquad L_{g,r}, \qquad U_{g,r}.
\]

The figure supplies the corresponding vector coordinates:

\[
y_{m,g,r}, \qquad y_{u,g,r}, \qquad y_{l,g,r}.
\]

The goal is to infer \(\mu_{g,t}, L_{g,t}, U_{g,t}\) for another visit \(t\).

---

# 5. Workflow overview

The recommended workflow is:

1. Verify what the figure represents statistically.
2. Identify all treatment series and all plotted visits.
3. Independently retrieve exact numerical reference values from the paper.
4. Convert or inspect the vectorized figure.
5. Identify vector colors, marker shapes, line paths, and error-bar paths for each treatment group.
6. Extract exact x- and y-coordinates for the relevant graphical objects.
7. Establish x-axis visit mapping.
8. Establish y-axis numerical calibration.
9. Infer point estimates at every plotted visit.
10. Infer upper and lower confidence limits at every plotted visit.
11. Convert confidence intervals to standard errors if needed.
12. Perform internal geometric and numerical validation.
13. Report results with an appropriate number of digits and an explicit statement that intermediate values were digitized/inferred from the vector figure.

Each step is detailed below.

---

# 6. Step 1 — Verify the statistical estimand and figure definition

Before digitization, read the figure caption and statistical methods.

Record:

- endpoint definition;
- whether outcome is change from baseline, percentage change, absolute value, risk, hazard, etc.;
- analysis population;
- estimand;
- model type;
- whether values are raw means or least-squares/model-adjusted means;
- confidence level;
- whether missing data are modeled or imputed;
- whether values after treatment discontinuation are included or excluded;
- treatment groups;
- planned visit times.

This prevents using a numerically correct anchor from the wrong analysis.

For longitudinal clinical-trial plots, this distinction is critical because a paper may report different estimates for different estimands at the same nominal week.

---

# 7. Step 2 — Identify treatment series and visit structure

From the legend and plot:

1. enumerate all treatment groups;
2. identify each group's color;
3. identify each group's marker shape;
4. identify line style;
5. identify error-bar style;
6. enumerate every plotted visit from the x-axis.

Create an internal map such as:

```text
Group A -> color X -> marker circle
Group B -> color Y -> marker triangle
Group C -> color Z -> marker diamond
Comparator -> gray -> open square -> dashed line
```

Do not rely on color alone if two colors are similar. Use color + marker + line style together.

---

# 8. Step 3 — Retrieve independent numerical anchors

For each treatment group, independently locate the exact reference point estimate and confidence interval from the article.

A final follow-up visit is often ideal because:

- it is frequently reported numerically in the Results section;
- confidence intervals are often given in the main text or tables;
- the same final visit is plotted in the longitudinal figure;
- all treatment groups may have values reported separately.

For each treatment group \(g\), record internally:

```text
reference_visit_g
reference_mean_g
reference_lower_CI_g
reference_upper_CI_g
source_location_g
estimand_match = yes/no
```

Do not proceed until the estimand and plotted series are confirmed to match.

### Important independence rule

If this skill is being used to obtain an independent readout, **do not preload previously digitized coordinates or previously inferred intermediate values**. Only preload the original article, figure, and legitimate textual/tabular reference values.

---

# 9. Step 4 — Obtain vector geometry

If working with a PDF, convert the relevant page to SVG or inspect the PDF drawing objects directly.

A typical command-line workflow is conceptually:

```text
PDF page -> SVG -> parse XML paths and transforms
```

The exact software is not important. Suitable tools may include PDF-to-SVG utilities, PDF graphics libraries, or direct PDF path extraction.

The goal is to recover:

- path definitions;
- stroke colors;
- fill colors;
- transformation matrices;
- marker paths;
- line paths;
- vertical error-bar stems;
- horizontal error-bar caps.

Do not rasterize first unless necessary.

---

# 10. Step 5 — Resolve transformed coordinates

SVG/PDF paths are often defined in local coordinates and then transformed.

For a standard affine SVG transformation matrix:

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

Apply the full transformation before comparing object locations.

Do not compare raw local path coordinates from two elements if they use different transformation matrices.

After transformation, all relevant coordinates should be expressed in the same page coordinate system.

---

# 11. Step 6 — Identify the longitudinal trajectory path

A longitudinal trajectory is often represented by one polyline/path per treatment group.

A typical vector path may contain a sequence such as:

```text
M x0 y0
L x1 y1
L x2 y2
...
L xK yK
```

After applying transformations, the ordered vertices provide the marker-center locations for successive visits if the plotting software uses the trajectory vertex as the marker center.

Confirm this by checking that:

- the vertex x-coordinates align with visit tick locations;
- the vertex y-coordinates align with marker centers;
- independent marker objects at one or more visits coincide with the line vertices.

If yes, the trajectory path is often the most precise way to extract all mean-marker centers at once.

---

# 12. Step 7 — Identify confidence-interval geometry

For each visit, confidence intervals may be encoded as several vector elements:

- an upper vertical stem segment;
- a lower vertical stem segment;
- a top horizontal cap;
- a bottom horizontal cap;
- a marker centered between them.

Extract the y-coordinate of the **centerline** of the top cap and bottom cap.

If the cap is a horizontal line, both cap endpoints should have the same y-coordinate. The exact x-coordinate is secondary; the cap's y-coordinate is what maps to the confidence limit.

Use the vertical stem endpoints as a cross-check. The stem should terminate at the cap centerline, up to numerical precision and stroke rendering conventions.

---

# 13. Step 8 — Map x-coordinate to follow-up time

Identify the x-coordinate associated with every visit.

Preferred method:

1. extract x-axis tick mark positions;
2. pair them with printed visit labels;
3. verify that treatment-marker x-coordinates coincide with those tick positions.

If x-axis spacing is irregular because the plot uses actual numerical time, use the affine x-axis calibration.

If plotted visits are categorical/equally spaced despite unequal numerical time gaps, do **not** infer time from horizontal distance. Instead, map each marker to the labeled visit by its exact x-position/order.

For the target figure, the task is not to interpolate between visits; it is to identify the already plotted discrete visits.

---

# 14. Step 9 — Calibrate the y-axis using known values

There are several valid calibration strategies. Use more than one when possible.

## 14.1 Strategy A: axis-tick calibration

If two or more y-axis tick coordinates and printed values can be extracted precisely, fit:

\[
V = a + b y.
\]

With two anchors \((y_1,V_1)\) and \((y_2,V_2)\):

\[
b = \frac{V_2-V_1}{y_2-y_1},
\]

\[
a = V_1 - b y_1.
\]

Then:

\[
\hat V(y)=a+by.
\]

This is a global plot calibration.

## 14.2 Strategy B: marker-to-marker reference calibration

When a treatment group has a known reference mean at visit \(r\) and another known value such as baseline zero at visit \(0\), use:

\[
\mu_{g,t}
=
\mu_{g,0}
+
(\mu_{g,r}-\mu_{g,0})
\frac{y_{m,g,t}-y_{m,g,0}}
{y_{m,g,r}-y_{m,g,0}}.
\]

For a change-from-baseline plot, \(\mu_{g,0}=0\) is often known by definition, giving:

\[
\mu_{g,t}
=
\mu_{g,r}
\frac{y_{m,g,t}-y_{m,g,0}}
{y_{m,g,r}-y_{m,g,0}}.
\]

This is particularly useful because it anchors the same graphical series to a known numerical endpoint.

## 14.3 Strategy C: reference mean plus known CI cap

If the reference visit supplies a known mean and upper confidence limit:

\[
s_u
=
\frac{U_{g,r}-\mu_{g,r}}
{y_{m,g,r}-y_{u,g,r}}.
\]

Similarly, using the lower confidence limit:

\[
s_l
=
\frac{\mu_{g,r}-L_{g,r}}
{y_{l,g,r}-y_{m,g,r}}.
\]

These scale factors convert graphical error-bar distance to real-world confidence-margin distance.

Because the y-axis is linear, \(s_u\) and \(s_l\) should be almost identical. Any meaningful discrepancy suggests one of:

- incorrect graphical-object identification;
- wrong reference CI;
- rounding in the published CI;
- transformation error;
- stroke-edge measurement instead of centerline measurement.

---

# 15. Step 10 — Infer point estimates at every plotted visit

For each treatment group and each plotted visit, infer the mean using one of the calibrated mappings.

Preferred approach:

- use the global y-axis affine mapping if confidently extracted;
- also compute the same mean using a treatment-specific known reference marker;
- compare the two estimates.

The estimates should agree to within a small numerical tolerance.

If they do not, investigate before reporting results.

Do not force agreement by manually adjusting coordinates.

---

# 16. Step 11 — Infer confidence limits at every plotted visit

For target visit \(t\), after obtaining \(\mu_{g,t}\), compute the graphical upper and lower margins:

\[
\Delta y_{u,g,t}=y_{m,g,t}-y_{u,g,t},
\]

\[
\Delta y_{l,g,t}=y_{l,g,t}-y_{m,g,t}.
\]

Then using reference-visit scale factors:

\[
U_{g,t}
=
\mu_{g,t}+s_u\Delta y_{u,g,t},
\]

\[
L_{g,t}
=
\mu_{g,t}-s_l\Delta y_{l,g,t}.
\]

Equivalently, if a global y-axis mapping \(V=a+by\) is available, simply transform the cap coordinates directly:

\[
U_{g,t}=a+b y_{u,g,t},
\]

\[
L_{g,t}=a+b y_{l,g,t},
\]

with the correct ordering determined by the sign of \(b\).

The two approaches should agree.

---

# 17. Step 12 — Convert confidence interval to standard error

If the plotted interval is a two-sided 95% confidence interval based on an approximately normal estimate, then:

\[
SE \approx \frac{U-L}{2\times1.96}.
\]

If the interval is symmetric around the mean:

\[
SE \approx \frac{U-\mu}{1.96}
\approx
\frac{\mu-L}{1.96}.
\]

However, do not assume normality without checking the article's analysis description.

For model-based least-squares means from a large-sample MMRM, the normal approximation is often reasonable, but the exact inferential method should be documented if available.

If the article uses a t critical value, transformed scale, bootstrap CI, Bayesian credible interval, or asymmetric construction, do not mechanically convert the CI to an SE using 1.96.

---

# 18. Redundant calibration is a feature, not a problem

A high-quality extraction should use multiple independent geometric checks.

Examples:

### Check 1: y-axis ticks

Use printed y-axis ticks to obtain \(a\) and \(b\).

### Check 2: known final-visit marker

Transform the known final-visit marker coordinate and verify that the reconstructed numerical mean matches the paper.

### Check 3: known final-visit CI caps

Transform the known top and bottom cap coordinates and verify that they reconstruct the reported confidence interval.

### Check 4: baseline

For a change-from-baseline outcome, confirm that baseline markers map to zero.

### Check 5: other treatment groups

Apply the same global y-axis calibration to all groups' known final-visit estimates. They should reproduce the independently reported values.

### Check 6: cap symmetry

When a reported CI is symmetric around a mean, compare graphical top and bottom distances from the marker center.

### Check 7: cross-method agreement

Compare:

- axis-derived values;
- reference-marker-derived values;
- CI-distance-derived values.

Do not report a highly precise digitized result unless these checks are mutually consistent.

---

# 19. Handling rounding in published reference values

Published anchor values may be rounded to one decimal place even though the underlying plotted coordinates were generated from a more precise model estimate.

Therefore, an apparent discrepancy of a few hundredths may be caused by article rounding rather than extraction error.

Recommended practice:

1. perform calculations using full vector-coordinate precision;
2. treat printed anchor values as rounded observations of the underlying estimate;
3. compare multiple anchors;
4. report inferred intermediate values at a precision justified by the plot and reference rounding.

For a plot whose anchor estimates are printed to one decimal place, reporting intermediate inferred values to **two decimals for technical documentation** may be reasonable, while reporting **one decimal place for practical extracted AgD use** is usually more defensible.

Do not imply that a digitized value is exact to more decimal places than the source supports.

---

# 20. Important object-identification pitfalls

## 20.1 Marker shape is not always a circle

Never describe every mean point as a dot or circle. Identify the actual marker shape used for that treatment group.

The numerical coordinate is the marker's logical center, regardless of shape.

## 20.2 Error-bar stroke width

A horizontal cap has visual thickness. Use the **vector line centerline**, not the top or bottom rendered edge of the stroke.

## 20.3 Overlapping line and marker objects

The trajectory path may run through the marker. This is useful: the path vertex can provide an independent center-coordinate check.

## 20.4 Open markers

For an open square or open circle, the fill may be white while the stroke carries the treatment color. Search using stroke attributes as well as fill attributes.

## 20.5 Error bars may use a different gray or opacity

The comparator's trajectory line, marker border, and CI error bars may not use exactly the same color value. Identify objects spatially and structurally, not only by a single RGB value.

## 20.6 PDF coordinate direction

Many PDF/SVG transformations reverse the vertical axis. Always infer direction empirically.

## 20.7 Broken x-axis

If the figure contains a break before another estimand or a separate endpoint display, do not treat post-break symbols as part of the same continuous x-axis.

## 20.8 Multiple estimands in the same panel

Some figures display one longitudinal estimand and also show a separate final-visit estimate from another estimand after an axis break. Do not use the wrong symbol as an anchor.

---

# 21. Special instructions for longitudinal treatment plots

When the goal is to recover every group's point estimate and CI at every plotted week:

1. Extract all visit x-coordinates first.
2. Extract the full mean trajectory for each group.
3. Extract all corresponding top and bottom error-bar cap coordinates.
4. Obtain independent final-visit mean and CI anchors for each treatment group from the article.
5. Establish a global y-axis calibration.
6. Verify all final-visit anchors against the calibration.
7. Infer all intermediate means.
8. Infer all intermediate upper CI limits.
9. Infer all intermediate lower CI limits.
10. Derive SEs if requested and statistically appropriate.
11. Produce a rectangular table with one row per treatment-by-visit combination.

Recommended output columns:

```text
treatment
week
mean
lower_95_ci
upper_95_ci
se
source_type
extraction_method
```

Suggested `source_type` values:

```text
reported_textually
reported_in_table
reported_on_figure
vector_inferred
```

Suggested `extraction_method` values:

```text
direct_report
axis_calibration
reference_marker_calibration
reference_CI_cap_calibration
```

---

# 22. Recommended algorithm for automated extraction

A robust AI/tool-assisted implementation can follow this pseudocode.

```text
INPUT:
    original PDF containing figure
    article main text
    relevant tables / supplementary appendix
    target figure identifier

A. READ STUDY CONTEXT
    identify endpoint
    identify estimand
    identify analysis model
    identify confidence level
    identify treatment groups
    identify visit times

B. RETRIEVE REFERENCE VALUES
    for each treatment group g:
        independently retrieve exact known reference mean and CI
        verify estimand/timepoint match

C. EXTRACT VECTOR FIGURE
    convert relevant PDF page to SVG or equivalent
    parse graphical elements
    resolve affine transforms

D. IDENTIFY SERIES
    map color + marker + line style to each group
    locate trajectory path
    locate marker objects
    locate CI caps/stems

E. IDENTIFY VISITS
    map x coordinates to labeled weeks

F. EXTRACT COORDINATES
    for every g,t:
        store y_mean[g,t]
        store y_upper_cap[g,t]
        store y_lower_cap[g,t]

G. CALIBRATE Y SCALE
    fit global affine y mapping using axis ticks
    independently derive reference-group mapping using known final values
    compare

H. VALIDATE REFERENCE VISIT
    for every treatment group:
        reconstructed final mean ~= published final mean
        reconstructed upper CI ~= published upper CI
        reconstructed lower CI ~= published lower CI

I. INFER ALL VISITS
    for every g,t:
        mean[g,t] = transform(y_mean[g,t])
        upper[g,t] = transform(y_upper_cap[g,t])
        lower[g,t] = transform(y_lower_cap[g,t])

J. DERIVE SE IF APPROPRIATE
    SE[g,t] = (upper-lower)/(2*1.96)

K. QUALITY CONTROL
    inspect monotonicity/plausibility
    inspect CI ordering
    compare redundant calibrations
    flag any inconsistent graphical object

OUTPUT:
    treatment-by-visit table
    concise extraction-method note
    validation diagnostics
```

---

# 23. Quality-control tolerances

Because vector geometry is precise but reported anchors may be rounded, use practical tolerances.

Suggested checks:

- reconstructed known anchor mean should match the reported value within expected publication rounding;
- reconstructed known CI bounds should match within expected publication rounding;
- global y-axis scale inferred from several anchors should be nearly constant;
- upper and lower pixel-to-value scale factors from a symmetric reference CI should be nearly equal;
- all visits for a given treatment should have x-coordinates matching the intended visit positions;
- no confidence bound should cross the point estimate in the wrong direction.

If one group fails validation while others pass, suspect incorrect object assignment rather than a global calibration failure.

---

# 24. When not to use this method

Do not use this method without modification when:

- the plot uses a logarithmic y-axis and the calibration is treated as linear;
- the plot uses a nonlinear transformed scale;
- the figure is only a low-resolution raster and vector geometry is unavailable;
- confidence bands are smooth ribbons with no visit-specific bounds and the desired estimate is not at a polygon vertex;
- jitter, dodge, or intentional horizontal displacement obscures visit identity;
- the plotted values were manually redrawn rather than generated from the analysis output;
- the known numerical anchor corresponds to a different estimand or population;
- the error bars represent something other than the claimed interval, such as standard error, standard deviation, or interquartile range.

For logarithmic or other transformed axes, calibrate in the transformed coordinate scale first and then back-transform.

---

# 25. Independence and anti-contamination rules

This skill is designed to support **independent repeated readouts** by different AI systems.

Therefore:

1. Do not embed prior AI-extracted intermediate coordinates in the skill file.
2. Do not embed prior AI-inferred intermediate visit values in the skill file.
3. Do not tell a new AI what numerical answers it is expected to recover at unlabeled visits.
4. Provide only the original source figure and legitimate article-reported anchors.
5. Require the new AI to identify the vector objects itself.
6. Require the new AI to reconstruct the calibration itself.
7. Require a validation step against the original reported anchor values.

The purpose is reproducibility through **methodological consistency**, not answer memorization.

---

# 26. Recommended result-reporting language

When reporting digitized results, distinguish explicitly between directly reported and vector-inferred values.

Example wording:

> Intermediate visit estimates were reconstructed from the vectorized published figure. Exact numerical values reported elsewhere in the article at a reference visit were used to calibrate the figure's vector-coordinate scale. Marker-center coordinates were used for point estimates, and the centerlines of the upper and lower error-bar caps were used for confidence-limit extraction. The resulting values should be regarded as high-precision figure-derived estimates rather than originally tabulated numerical results.

Do not describe an inferred value as "reported by the trial" if the exact number was not textually or tabularly reported.

---

# 27. Application-specific instructions for a Figure 1B–type plot

When applying this skill to a longitudinal percent-change-in-body-weight panel with multiple treatment groups and placebo:

1. Confirm that the panel displays the intended efficacy estimand and analysis method.
2. Identify every treatment trajectory and marker shape from the legend.
3. Identify every plotted week from the x-axis.
4. Retrieve the exact final-visit efficacy-estimand point estimate and 95% CI for **each treatment group separately** from the main text, tables, or other authoritative article content.
5. Do not substitute a pooled analysis if separate treatment-group estimates are plotted.
6. Do not substitute a different estimand shown elsewhere in the same figure.
7. Recover the vector coordinates of:
   - every mean marker center;
   - every upper CI cap centerline;
   - every lower CI cap centerline.
8. Calibrate using both the y-axis ticks and the independently reported final-visit values.
9. Validate all final-visit treatment groups before inferring intermediate visits.
10. Return point estimate, lower 95% CI, upper 95% CI, and SE for every treatment group at every plotted week.
11. Preserve enough internal precision to allow reproducibility, but round the final practical readout according to source precision.

---

# 28. Final checklist for the AI

Before returning results, verify all of the following:

- [ ] I confirmed the figure's endpoint, estimand, and model.
- [ ] I independently retrieved the correct reference means and confidence intervals.
- [ ] I did not reuse prior AI digitization results.
- [ ] I verified that the PDF/figure is vectorized.
- [ ] I identified each treatment group by color, marker shape, and line style.
- [ ] I extracted transformed page coordinates, not incomparable local coordinates.
- [ ] I used marker centers for point estimates.
- [ ] I used error-bar cap centerlines for confidence limits.
- [ ] I mapped marker x-coordinates to the correct visits.
- [ ] I calibrated the y-axis numerically.
- [ ] I validated the calibration against all available known reference values.
- [ ] I inferred intermediate point estimates from vector coordinates.
- [ ] I inferred intermediate CI limits from vector coordinates.
- [ ] I derived SE only if the confidence-interval construction permits it.
- [ ] I clearly distinguished direct reported values from vector-inferred values.
- [ ] I did not overstate numerical precision.

---

# 29. Conceptual summary

The fundamental procedure is:

\[
\boxed{\text{known numerical value}}
\longleftrightarrow
\boxed{\text{known vector coordinate}}
\]

which establishes a graphical-to-numerical calibration. Once calibrated:

\[
\boxed{\text{unknown vector coordinate}}
\longrightarrow
\boxed{\text{inferred numerical value}}.
\]

For point estimates, the relevant coordinate is the **marker center**.

For confidence limits, the relevant coordinates are the **upper and lower error-bar cap centerlines**.

For longitudinal plots, the same calibrated mapping can be applied systematically to every treatment group and every plotted follow-up visit.

The method is especially powerful when the source is a true vector PDF because it replaces subjective visual approximation with high-resolution geometric reconstruction while still remaining anchored to independently reported numerical results from the publication.
