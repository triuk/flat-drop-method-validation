# Implementation specification for `drop-analysis`

This is the **authoritative implementation target** for the flat-drop image-analysis workflow. It is based on the current scientific draft and on validation against a complete exported project containing one dry apparatus reference plus 23 frames acquired while water volume was progressively increased.

Do not silently replace the rules below with assumptions from the existing `needle` analysis mode.

## 1. Supported input project

The validated source project has:

```text
format: drop-analysis-project
schemaVersion: 11
reference.file: reference/apparatus_reference.png
frames[*].file: frames/frame_NNN.png
frames[*].offsetSeconds: acquisition time relative to burst start
```

The current example project stores `analysis.geometryMode = "needle"` and empty `needleFits`. Those fields are **not flat-drop ground truth**. Flat-drop results must be stored separately and must not depend on existing needle-fit values.

## 2. Acquisition model and purpose of the filling sequence

1. Acquire one **dry reference frame** before liquid is added.
2. Keep the apparatus/camera nominally fixed.
3. Add liquid slowly and acquire a time-ordered sequence that intentionally passes through:

```text
underfilled / tangent not yet vertical
        ↓
near-vertical region
        ↓
ideal vertical-tangent state
        ↓
overfilled relative to the ideal state
```

4. Preserve frame timestamps (`offsetSeconds`).

The sequence has **two purposes**, both of which must be supported by the software:

- locate the optimal state `t0`, where the side profile at the rim is closest to a vertical tangent;
- quantify robustness around `t0` by analyzing valid neighboring frames before and after the zero crossing.

Therefore, the implementation must **not reduce the sequence to one best frame**. The closest-to-zero frame may be highlighted as the nominal optimum, but valid neighboring frames are intentional experimental data and must remain available for robustness analysis.

Even with a nominally fixed camera, registration is required: the validated project shows a measurable image-coordinate offset between the dry reference and the liquid burst.

## 3. Coordinate definitions

Use image coordinates only as intermediate quantities.

After registering each liquid frame to the dry reference and rectifying the rim line, define:

```text
z = 0                 registered physical rim plane
z > 0                 upward into the drop
x                      image horizontal coordinate
x_contact              common pinned contact position for the analyzed sequence
q = x_contact - x      inward radial displacement on the right side
```

The physical `Rphi` used in the Young-Laplace formula remains a mechanically/calibrationally defined quantity. Do not infer physical `Rphi` from the fitted image intercept of the liquid profile.

## 4. Dry-reference processing

### 4.1 Rim plane

Detect the upper basin/air silhouette over a wide horizontal span by the strong vertical intensity transition.

Recommended tested procedure:

1. light Gaussian smoothing only for edge localization,
2. compute vertical intensity gradient,
3. locate the strong air-to-basin edge column-wise,
4. refine each edge location with three-point quadratic subpixel interpolation,
5. robustly fit a line `y_rim(x) = p0 + p1*x`,
6. use that line as the physical `z = 0` plane after registration.

On the current dry reference the fitted line was approximately

```text
y(x) = 966.627 + 0.000363 x
```

with residual SD about `0.34 px` for the selected strong-edge samples. These are regression-test values for this fixture, not universal constants.

### 4.2 What NOT to derive from the dry silhouette

Do **not** treat the outer visible silhouette corner as the liquid contact x-coordinate.

In the validated apparatus, the liquid-air interface starts on an inner sharp edge and a small ledge continues to the right. The dry silhouette does not directly identify the liquid contact location with sufficient confidence.

Therefore:

- dry reference defines the **rim height/plane**, orientation, calibration and static registration ROI;
- liquid frames define the actual **contact x-position**.

### 4.3 Optical lower-edge warning

Do not infer rim height from the bright lower ledge visible in strongly illuminated liquid frames. Large drops can over-illuminate this region and make the rim appear lower than it is.

## 5. Registration of liquid frames

Register every liquid frame to a static ROI from the dry reference before using the dry rim plane.

Do not use the liquid-air interface for registration.

For the current batch, cross-correlation of gradient profiles from a static background/grid ROI produced reference-to-frame translations of approximately:

```text
dx = +1.74 ... +2.22 px
dy = +1.60 ... +1.81 px
```

Implementation requirements:

- translation registration is mandatory;
- keep the method modular so rotation/affine correction can be added if needed later;
- save the estimated transform and registration quality metric per frame;
- registration failure must produce QC failure, not a silent result;
- thresholds remain configurable until multiple independent sequences are validated.

A robust equivalent to the tested gradient-profile cross-correlation is acceptable only if it reproduces the same fixture behavior.

## 6. Plateau height `h`

Plateau estimation and side-curvature estimation are independent tasks.

Use a wide ROI well away from the right curved rim region.

Recommended procedure:

1. detect the upper liquid-air transition using the vertical intensity gradient,
2. apply subpixel peak localization,
3. fit a line to the plateau interface if needed,
4. calculate `h` from the plateau level relative to the **registered dry rim plane**.

On the validated filling batch, image-space `h` increased monotonically from approximately:

```text
238.78 px  (frame 001)
281.93 px  (frame 023)
```

This monotonicity is a useful sequence-level QC diagnostic for a monotonic filling experiment, but not a universal requirement for arbitrary sequences.

## 7. Side liquid-air interface extraction

Use a right-side ROI around the liquid-air boundary.

For each image row:

1. compute the horizontal intensity gradient,
2. locate the positive gradient maximum corresponding to liquid -> background,
3. refine the peak with three-point quadratic subpixel interpolation,
4. store the resulting interface point.

### Selected default

**Horizontal gradient maximum + three-point quadratic subpixel interpolation.**

This was preferred over integer peak, gradient centroid and logistic intensity-edge fitting on the tested image because it retained good fit stability while providing subpixel coordinates; the logistic fit was less stable on the tested data.

Keep the extractor modular so alternatives can be compared diagnostically, but do not use the logistic fit as the current default.

## 8. Contact-position estimation and pinning QC

The contact x-coordinate must be obtained from liquid frames, not from the outer dry silhouette.

For each registered liquid frame, over a short configurable near-rim interval, fit

```text
x(z) = xc_i + s_i*z + c2_i*z^2
```

with a **free intercept**.

`xc_i` is the extrapolated contact position at the registered physical rim plane `z = 0`.

Sequence-level procedure:

1. collect all `xc_i`,
2. estimate the dominant common cluster robustly,
3. define common pinned contact position `x_contact` from that cluster,
4. flag frames outside the cluster as `contact_position_outlier`.

For the current batch, using a provisional short fit interval of about `10..60 px`:

```text
robust median contact x in reference coordinates = 2326.923 px
MAD                                             = 0.485 px
frame 023 offset from median                    = +10.265 px
frame 023 offset                                = 21.17 MAD
```

The physical cause of the final-frame discontinuity is intentionally **not assumed** by the algorithm. Detect and report the discontinuity; do not label it as depinning/overflow unless separately established.

MAD-based outlier detection is recommended. The exact multiplier must remain configurable; `5 MAD` separates frame 023 from frames 001-022 in the current fixture but is not a universal threshold.

## 9. Objective vertical-tangent / `t0` detection

After determining common pinned `x_contact`, fit each valid candidate frame over a short near-rim interval using

```text
q = c1*z + a*z^2
q = x_contact - x
```

The coefficient `c1` is the directly measured rim-slope diagnostic in these coordinates.

```text
c1 = 0   <=>   vertical tangent at the rim plane
```

Use it to:

1. identify the frame closest to the ideal state;
2. detect a sign change when the filling sequence passes through the optimum;
3. interpolate the zero crossing using frame timestamps;
4. rank valid frames on both sides of `t0` by `abs(c1)` and/or time distance from `t0`.

For the validated batch:

```text
frame 006: c1 ≈ +0.0120
frame 007: c1 ≈ -0.0047
linear zero crossing: t0 ≈ 11.95 s
```

The short-fit interval remains configurable. The current demonstration used approximately `z = 10..60 px`; this is not universal.

### Important interpretation

A nonzero but small `c1` is **not automatically a failure**. Such frames are deliberately present to test robustness to slightly imperfect filling.

Do not convert `c1` to an angular deviation `DeltaPsi` until the coordinate/angle convention is explicitly verified against the scientific definition. Store and report `c1` directly in the current implementation.

## 10. Curvature fitting: do not force the radial intercept

The curvature fit must **not** force the fitted radial intercept to equal a preselected image `x0`.

Curvature depends on derivatives of the profile; a fixed image intercept makes `Rtheta` unnecessarily sensitive to small contact-coordinate errors.

### 10.1 Article-aligned near-vertical model

For frames in the neighborhood of the vertical-tangent state, fit

```text
x(z) = c0 + c2*z^2
```

where `c0` is free.

Then

```text
Rtheta = 1 / (2*abs(c2))
```

in image units before px/mm conversion.

This is the image-coordinate equivalent of the local parabolic model in the scientific draft; the additive coordinate offset does not affect curvature.

### 10.2 General local diagnostic model

Also calculate

```text
x(z) = c0 + c1*z + c2*z^2
```

and curvature at `z = 0`

```text
Rtheta_general = (1 + c1^2)^(3/2) / (2*abs(c2))
```

Use `Rtheta_general` as a diagnostic/cross-check and for studying small tangent deviations. Do not silently substitute it for the article-aligned reported value unless the scientific method definition is explicitly updated.

## 11. Adaptive curvature-fit window

Do not use one fixed profile length.

For every frame included in the `t0` robustness neighborhood, sweep progressively increasing `z_max`.

For the article-aligned local fit compare:

```text
x = c0 + c2*z^2
x = c0 + c2*z^2 + c4*z^4
```

Estimate `sigma_c4` and accept a candidate interval with respect to the higher-order term only when

```text
abs(c4) < 2*sigma_c4
```

This implements the scientific draft criterion `|b| < 2 sigma_b` while keeping the irrelevant absolute radial intercept free.

Then:

1. identify contiguous accepted `z_max` blocks,
2. evaluate `Rtheta` stability within each block,
3. choose a final window only from a block satisfying both the higher-order criterion and the stability criterion.

Still configurable / not universal:

- `z_min`,
- short contact/tangent diagnostic interval,
- `z_max` increment,
- minimum fit points,
- minimum accepted-block length,
- `Rtheta` stability threshold,
- contact-cluster MAD multiplier,
- registration-quality and registration-shift thresholds.

Do not hide these as undocumented constants.

## 12. Robustness analysis around `t0`

This is a required sequence-level output, not an optional visualization.

After contact-position QC and tangent estimation:

1. determine `t0` or the closest-to-zero frame;
2. define a configurable neighborhood around `t0` using one of these explicitly recorded criteria:
   - frame count around `t0`,
   - time window around `t0`, or
   - maximum `abs(c1)`;
3. keep valid frames on **both sides** of the optimum;
4. perform plateau and curvature analysis for each retained frame;
5. report how `h`, `Rtheta` and, once physical calibration is enabled, derived capillary length / surface tension vary across the neighborhood.

Recommended plots/tables:

```text
c1 vs time
Rtheta vs time - t0
Rtheta vs c1
h vs time - t0
later: gamma vs time - t0
gamma vs c1
```

The closest-to-zero frame is the nominal optimum and should be highlighted, but it must not be the only analyzed result.

The purpose is to provide direct experimental evidence for the claimed robustness when the filling is only approximately optimal.

## 13. Processing order

Implement the pipeline in this order:

```text
load project + dry reference + timestamps
        ↓
fit dry rim plane and calibration
        ↓
register every liquid frame to dry reference
        ↓
extract plateau → h
        ↓
extract right liquid-air interface
        ↓
per-frame free-intercept contact fit → xc_i
        ↓
robust sequence contact cluster → x_contact
        ↓
flag contact-position outliers
        ↓
short fixed-contact tangent fit → c1
        ↓
find c1 zero crossing / closest frame → t0
        ↓
define configurable valid neighborhood around t0
        ↓
curvature fit EVERY retained frame
  with free radial intercept
        ↓
adaptive z_max + fourth-order criterion + Rtheta stability
        ↓
per-frame h + Rtheta + QC
        ↓
sequence robustness report
  before / at / after t0
```

## 14. Required per-frame outputs

Save at least:

- frame index and timestamp,
- registration transform and quality,
- registered rim-plane parameters,
- plateau level and `h`,
- contact intercept `xc_i`,
- contact-cluster residual / robust normalized residual,
- contact-position QC state,
- tangent coefficient `c1` and its uncertainty,
- signed time offset from `t0`,
- whether the frame belongs to the selected robustness neighborhood,
- selected curvature model,
- `Rtheta` and uncertainty,
- optional `Rtheta_general`,
- selected `z_min` and `z_max`,
- quadratic-fit RMSE,
- `c4`, `sigma_c4`, `abs(c4)/sigma_c4`,
- overall pass/fail state and explicit failure reason.

## 15. Required sequence-level outputs

Save at least:

- common contact-position estimate,
- robust contact spread (MAD or equivalent),
- detected `t0` or closest frame if no sign change exists,
- list of contact-position outliers,
- ranking of valid frames by `abs(c1)` / distance from `t0`,
- explicit definition of the selected robustness neighborhood,
- summary table for all retained frames,
- plots of `h`, contact position, `c1` and `Rtheta` versus frame/time,
- `Rtheta` versus `c1`,
- later, when physical calibration is active, equivalent robustness plots for derived surface tension.

## 16. Diagnostic overlays

During method validation, every accepted/rejected result must be visually auditable.

Per-frame overlay should show:

- registered rim plane,
- extracted interface points,
- extrapolated/common contact position,
- short tangent-fit interval,
- selected curvature-fit interval,
- fitted local curve,
- plateau reference,
- key numerical values and QC flags.

A numerical result without an overlay should not be the default validation output.

## 17. Regression fixture values from the current 23-frame project

Use `results/batch-validation.csv` as a regression fixture.

Expected qualitative and approximate numerical behaviors:

- registration finds a roughly 2 px dry-reference offset, not zero;
- `h` rises monotonically from about 238.8 to 281.9 px;
- contact positions for frames 001-022 form one robust cluster;
- frame 023 is a strong contact-position outlier;
- tangent coefficient crosses zero between frames 006 and 007;
- interpolated `t0` is approximately 11.95 s;
- frames on both sides of this zero crossing must remain available for robustness analysis rather than being discarded solely because `c1 != 0`.

If a new implementation does not reproduce these qualitative and approximate behaviors, inspect edge extraction, registration and coordinate conventions before trusting curvature output.

## 18. Scientific quantities not yet finalized here

The following remain outside the fully validated image-only implementation target:

- definitive px/mm calibration for this source project,
- final uncertainty propagation to physical `Rtheta`, `h`, capillary length and surface tension,
- universal `Rtheta` stability threshold,
- universal contact-outlier threshold,
- universal definition of the `t0` robustness-neighborhood width,
- repeatability across independent filling sequences.

The scientific draft defines the target flat-drop relation, the local-parabolic model and the `|b| < 2 sigma_b` criterion. This specification defines how the currently validated image processing should supply geometrical inputs and how the intentionally underfilled-to-overfilled sequence should be used to test robustness around the vertical-tangent optimum.
