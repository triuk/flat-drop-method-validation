# Implementation specification for `drop-analysis`

This is the **authoritative implementation target** for the flat-drop image-analysis workflow. It is based on the current scientific draft and on validation against a complete exported project containing one dry apparatus reference plus 23 filling frames.

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

## 2. Acquisition model

1. Acquire one **dry reference frame** before liquid is added.
2. Keep the apparatus/camera nominally fixed.
3. Acquire a time-ordered sequence while liquid is added slowly.
4. Preserve frame timestamps (`offsetSeconds`).

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

### Tested registration signal

For the current batch, cross-correlation of gradient profiles from a static background/grid ROI produced reference-to-frame translations of approximately:

```text
dx = +1.74 ... +2.22 px
dy = +1.60 ... +1.81 px
```

with strong profile correlations in the tested ROI.

This proves that the dry geometry cannot simply be copied to identical pixel coordinates in later frames.

### Implementation requirements

- translation registration is mandatory;
- keep the method modular so rotation/affine correction can be added if a future setup requires it;
- save the estimated transform and registration quality metric per frame;
- registration failure must produce QC failure, not a silent result;
- thresholds must remain configurable until multiple independent sequences are validated.

A robust equivalent to the tested gradient-profile cross-correlation is acceptable, but it must be demonstrated on the same fixture before replacing it.

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

This monotonicity can be retained as a useful sequence-level QC diagnostic for a monotonic filling experiment, but it is not a universal requirement for arbitrary input sequences.

## 7. Side liquid-air interface extraction

Use a right-side ROI around the liquid-air boundary.

For each image row:

1. compute the horizontal intensity gradient,
2. locate the positive gradient maximum corresponding to liquid -> background,
3. refine the peak with three-point quadratic subpixel interpolation,
4. store the resulting interface point.

### Selected method

Default to:

**horizontal gradient maximum + three-point quadratic subpixel interpolation**.

This was preferred over integer peak, gradient centroid and logistic intensity-edge fitting on the tested image because it retained good fit stability while providing subpixel coordinates; the logistic fit was less stable on the tested data.

Keep the extractor modular so alternatives can be compared in diagnostics, but do not use the logistic fit as the current default.

## 8. Contact-position estimation and pinning QC

The contact x-coordinate must be obtained from liquid frames, not from the outer dry silhouette.

For each registered liquid frame, over a short configurable near-rim interval, fit the diagnostic model

```text
x(z) = xc_i + s_i*z + c2_i*z^2
```

with a **free intercept**.

`xc_i` is the extrapolated contact position at the registered physical rim plane `z = 0`.

### Sequence-level contact reference

1. collect all `xc_i`,
2. estimate the dominant common cluster robustly,
3. define the common pinned contact position `x_contact` from that cluster,
4. flag frames outside the cluster as `contact_position_outlier`.

For the current batch, using a provisional short fit interval of about `10..60 px`:

```text
robust median contact x in reference coordinates = 2326.923 px
MAD                                             = 0.485 px
frame 023 offset from median                    = +10.265 px
frame 023 offset                                = 21.17 MAD
```

The physical cause of the final-frame discontinuity is intentionally **not assumed** by the algorithm. The software should detect and report the discontinuity, not label it as depinning/overflow unless separately established.

### Threshold

MAD-based outlier detection is recommended because it is robust to isolated contact-position jumps. The exact multiplier must be configurable; a provisional factor of 5 MAD separates frame 023 from frames 001-022 in the current fixture, but this is not yet a universal threshold.

## 9. Objective vertical-tangent / `t0` detection

After determining the common pinned `x_contact`, fit each candidate frame over a short near-rim interval using

```text
q = c1*z + a*z^2
q = x_contact - x
```

The coefficient `c1` is the rim slope diagnostic in the chosen coordinates.

```text
c1 = 0   <=>   vertical tangent at the rim plane
```

Use this to identify the frame closest to the ideal state and, when a sign change exists, interpolate the zero crossing using frame timestamps.

For the validated batch:

```text
frame 006: c1 ≈ +0.0120
frame 007: c1 ≈ -0.0047
linear zero crossing: t0 ≈ 11.95 s
```

This replaces subjective frame selection with an objective diagnostic.

The short-fit interval remains configurable. The current demonstration used approximately `z = 10..60 px`; this range is not yet universal.

## 10. Curvature fitting: do not force the radial intercept

A key correction from the first single-image prototype is that the curvature fit must **not** force the fitted radial intercept to equal a preselected image `x0`.

Curvature depends on derivatives of the profile; a fixed image intercept makes `Rtheta` unnecessarily sensitive to small contact-coordinate errors.

### 10.1 Article-aligned near-vertical model

For frames accepted as near the vertical-tangent state, fit

```text
x(z) = c0 + c2*z^2
```

where `c0` is a free image intercept.

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

and the curvature at `z = 0`

```text
Rtheta_general = (1 + c1^2)^(3/2) / (2*abs(c2))
```

This follows directly from the plane-curve curvature formula.

Use `Rtheta_general` as a diagnostic/cross-check and for studying small tangent deviations. Do not silently substitute it for the article-aligned reported value unless the scientific method definition is explicitly updated.

## 11. Adaptive curvature-fit window

Do not use one fixed profile length.

After frame/contact/tangent QC, sweep progressively increasing `z_max`.

For the article-aligned near-vertical fit, compare:

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

### Still configurable / not universal

- `z_min`,
- short contact/tangent diagnostic interval,
- `z_max` increment,
- minimum fit points,
- minimum accepted-block length,
- `Rtheta` stability threshold,
- contact-cluster MAD multiplier,
- registration-quality and registration-shift thresholds.

Do not hide these as undocumented constants.

## 12. Processing order

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
find c1 ≈ 0 / timestamp zero crossing → t0
        ↓
select near-vertical candidate frames
        ↓
curvature fits with FREE radial intercept
        ↓
adaptive z_max + fourth-order criterion + Rtheta stability
        ↓
Rtheta + h + QC + diagnostic overlays
```

## 13. Required per-frame outputs

Save at least:

- frame index and timestamp,
- registration transform and quality,
- registered rim-plane parameters,
- plateau level and `h`,
- contact intercept `xc_i`,
- contact-cluster residual / robust normalized residual,
- contact-position QC state,
- tangent coefficient `c1` and its uncertainty,
- selected curvature model,
- `Rtheta` and uncertainty,
- optional `Rtheta_general`,
- selected `z_min` and `z_max`,
- quadratic-fit RMSE,
- `c4`, `sigma_c4`, `abs(c4)/sigma_c4`,
- overall pass/fail state and explicit failure reason.

## 14. Required sequence-level outputs

Save at least:

- common contact-position estimate,
- robust contact spread (MAD or equivalent),
- detected `t0` or closest frame if no sign change exists,
- list of contact-position outliers,
- list/ranking of near-vertical frames,
- summary plots of `h`, contact position and `c1` versus frame/time.

## 15. Diagnostic overlays

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

## 16. Regression fixture values from the current 23-frame project

Use the repository file `results/batch-validation.csv` as a regression fixture for the current implementation.

Important expected behaviors, subject to small numerical differences from implementation details:

- registration finds a roughly 2 px dry-reference offset, not zero;
- `h` rises monotonically from about 238.8 to 281.9 px;
- contact positions for frames 001-022 form one robust cluster;
- frame 023 is a strong contact-position outlier;
- the tangent coefficient crosses zero between frames 006 and 007;
- interpolated `t0` is approximately 11.95 s.

If a new implementation does not reproduce these qualitative and approximate numerical behaviors, inspect its edge extraction, registration and coordinate conventions before trusting curvature output.

## 17. Scientific quantities not yet finalized here

The following remain outside the validated image-only implementation target:

- definitive px/mm calibration for this source project,
- final uncertainty propagation to physical `Rtheta`, `h`, capillary length and surface tension,
- universal `Rtheta` stability threshold,
- universal contact-outlier threshold,
- repeatability across independent filling sequences.

The scientific draft defines the target flat-drop relation and the `|b| < 2 sigma_b` local-parabolic criterion; this implementation specification defines how the currently validated image processing should supply its geometrical inputs.
