# Implementation specification for `drop-analysis`

This document is the current implementation target derived from the validation work in this repository. It is intentionally narrower than the scientific article: it specifies only the image-processing and local-fit pipeline that is currently supported by the tested data.

## 1. Acquisition model

Use one fixed camera position for the complete measurement sequence.

1. Acquire a **dry reference frame** before liquid is added.
2. Keep the camera and dish fixed.
3. Acquire subsequent liquid frames while the dish is filled slowly.

The dry frame defines static dish geometry. Liquid frames define only quantities related to the liquid profile.

## 2. Static geometry from the dry frame

Determine and store:

- rim line and its image orientation,
- radial rim position,
- rim/contact reference point `(x0, y0)`,
- spatial calibration in px/mm,
- a static dish ROI suitable for registration checks.

Rectify the coordinate system so that the rim line is horizontal and corresponds to `z = 0`.

### Critical rule

Do **not** infer rim height from the bright lower ledge visible in strongly illuminated liquid frames. In the current optical setup, a large drop can over-illuminate the lower part of the rim and make the apparent rim position look lower than the physical rim. `y0` must correspond to the physical level at which the liquid-air interface starts.

## 3. Registration

For every liquid frame:

- compare a static dish ROI against the dry reference,
- estimate any residual translation/rotation,
- either correct it or reject the frame if displacement exceeds a configurable tolerance.

Do not estimate registration from the liquid-air interface.

## 4. Plateau height

Use an ROI well away from the curved rim region.

Recommended procedure:

1. detect the liquid-air transition by the vertical intensity gradient,
2. refine the interface position subpixel-wise,
3. fit a line to the plateau interface,
4. determine `h` relative to the fixed rim reference `z = 0`.

Plateau estimation and side-curvature estimation must remain separate operations.

## 5. Side-interface extraction

Use a right-side ROI around the curved liquid-air interface.

For each image row:

1. compute the horizontal intensity gradient,
2. locate the positive gradient maximum corresponding to the liquid-air transition,
3. apply three-point quadratic interpolation around the gradient maximum,
4. store the resulting subpixel interface coordinate.

### Selected method

**Horizontal gradient maximum + three-point quadratic subpixel interpolation.**

Reason for selection on `frame_046`:

- lowest RMSE among the tested subpixel approaches,
- fit stability close to the integer gradient maximum,
- avoids restriction to integer-pixel interface positions,
- gradient centroid did not improve the result,
- logistic intensity-edge fitting was less stable on the tested image.

Do not use the logistic edge fit as the default method for the current dataset.

## 6. Local coordinates

After rectification, use

```text
q = x0 - x
z = y0 - y
```

where `(x0, y0)` is fixed from the static geometry calibration.

Do not allow the production curvature fit to redefine `(x0, y0)`.

## 7. Curvature model

For a candidate local interval, fit

```text
q = a z²
```

and calculate

```text
Rtheta = 1 / (2a)
```

Also fit the diagnostic extended model

```text
q = a z² + b z⁴
```

and estimate the standard error `sigma_b` of `b`.

## 8. Adaptive fit-window selection

Do not use a fixed fit length.

For progressively increasing `z_max`:

1. fit the quadratic model,
2. fit the quadratic + fourth-order model,
3. calculate `Rtheta`, its fit uncertainty, RMSE, `b`, and `sigma_b`,
4. mark the interval as locally parabolic only when

```text
abs(b) < 2 * sigma_b
```

5. identify contiguous accepted blocks,
6. within accepted blocks evaluate the stability of `Rtheta`,
7. choose the final interval only from a block satisfying both conditions.

### Open parameter

The numerical definition of **Rtheta stability** has not yet been validated on enough frames. Implement it as a configurable criterion, not as a hidden hard-coded constant.

Likewise keep the following configurable:

- minimum distance `z_min` from the rim,
- `z_max` increment,
- minimum number of fit points,
- minimum length of an accepted contiguous block,
- maximum allowed registration shift,
- Rtheta-stability threshold.

The current demonstration used `z_min = 10 px`; this is not yet a validated universal value.

## 9. Diagnostics and QC

For each processed frame, save at least:

- `h`,
- `Rtheta`,
- uncertainty estimate of `Rtheta`,
- selected `z_min` and `z_max`,
- quadratic-fit RMSE,
- `b`, `sigma_b`, and `abs(b)/sigma_b`,
- registration shift,
- pass/fail state and failure reason.

Generate a diagnostic overlay containing:

- fixed rim reference,
- extracted interface points,
- selected fitting interval,
- fitted local parabola,
- plateau reference,
- key numerical fit values.

A numerical result without a diagnostic overlay should not be the default output during method validation.

## 10. Current validation result on `frame_046`

With the provisional image reference used during validation, the longest contiguous block satisfying the explicit fourth-order criterion was

```text
z_max = 90, 95, 100, 105 px
```

with

```text
median Rtheta = 118.256 px
mean Rtheta   = 118.231 px
SD            = 0.583 px
full range    = 1.354 px
relative full range = 1.145 %
```

These values validate the algorithmic behavior only. They are not a calibrated surface-tension result.

## 11. Required next validation steps

Before treating the pipeline as final:

1. validate automatic `(x0, y0)` determination on a true dry frame,
2. validate registration across a complete filling sequence,
3. define and test the numerical Rtheta-stability criterion,
4. test `z_min` sensitivity across multiple frames,
5. validate px/mm conversion and uncertainty propagation,
6. validate repeatability across independent filling sequences.
