---
topic: Intrinsic vs extrinsic rotation composition, and what URDF RPY actually means
status: draft
layer: math
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# Intrinsic vs extrinsic rotation composition

**One-line idea:**
Composing rotations about axes that move with the object ("intrinsic"/body-fixed) versus axes that stay fixed to a parent frame ("extrinsic"/fixed-axis) are genuinely different operations with opposite matrix-multiplication ordering conventions, and URDF's `rpy` is specifically fixed-axis (extrinsic) XYZ about the **parent** frame — not three independent protractor readings taken about the child object's own axes.

**Why it exists:**
Physically measuring or reasoning about a mounted sensor's orientation (e.g. "I tilted it X degrees, then twisted it Y degrees") only converts correctly into a URDF `rpy` triple if you know which rotation convention your measurements implicitly used — getting this wrong silently produces a plausible-looking but wrong orientation.

**Math / mechanics:**
- **URDF `rpy` is fixed-axis (extrinsic) XYZ, applied about the parent frame.** Starting with the child frame aligned to the parent, rotate about the parent's X axis, then the parent's Y axis, then the parent's Z axis. These three numbers are one parameterization of a single combined orientation — not three independently-measurable angles read off the child body as it's being rotated.
- **Intrinsic (body-fixed) composition:** each rotation is about the object's *own current* axis — i.e. the axis has already been carried along by every prior rotation, so which physical axis "Z" refers to depends on rotation history. For a sequence applied in chronological order (mount at some base orientation \(A\), then rotate about the object's own current Z, then about its own current (now-different) X), the matrix product reads **left to right** matching chronological order when each new rotation is appended on the right: \(A \cdot R_z \cdot R_x\). This follows from the conjugation identity \( \text{Rot}(A \cdot \hat{n}, \theta) \cdot A = (A R A^{-1}) A = A R \) — a rotation about an axis that's been carried along by \(A\) is equivalent to appending the original-frame rotation matrix on the right of \(A\).
- **Extrinsic (fixed-axis) composition:** each rotation is about the **parent/world's** axes, which never move regardless of how the object has already been reoriented. A new fixed-axis rotation multiplies on the **left**: first \(R_x\), then \(R_z \cdot R_x\) for a second step about the fixed Z. Chronological order reads **right to left** in this form — the opposite reading direction from the intrinsic case, for what can otherwise look like "the same" sequence of rotations.
- **In both conventions, vectors transform right-to-left through the matrix product** (ordinary function composition: \(R_2 (R_1 v)\) means apply \(R_1\) first) — that is a separate question from "which side does a new *chronological* rotation get appended on," and the two conventions disagree only on the latter, not on the former.
- **Fixed axes never moving does NOT mean order is irrelevant for extrinsic rotations.** The object's orientation still changes between steps even though the reference axes don't — so two fixed-axis 90° rotations applied in opposite orders still send the same starting vector to different final positions (e.g. rotate about fixed-X then fixed-Z vs. fixed-Z then fixed-X on the same starting vector `[0,0,1]` lands at different final positions). "Axis independence" and "order independence" are separate claims; only the former is true for extrinsic rotations.
- **Two genuine commuting exceptions:** rotations about the *same* axis always commute and simply add (\(R_z(10°) \cdot R_z(20°) = R_z(30°)\)); and small-angle rotations approximately commute (two ~1° rotations applied in opposite orders differ by only a tiny amount, on the order of the square of the angle in radians) — which is why a small correction twist is forgiving about exactly where it's inserted in a composition, while a large tilt angle is not.

**Code:**
```python
import numpy as np
from scipy.spatial.transform import Rotation as R

# Example: a camera mounted with a known base alignment, then a known tilt
# (about its own current Z/"up" axis) and a small twist (about its own
# current X/"look" axis) -- composed INTRINSICALLY (body-fixed), each new
# rotation appended on the right:
R_align = np.array([[0., 0., 1.],
                     [1., 0., 0.],
                     [0., 1., 0.]])   # e.g. lens along +Y, housing-up along +X

prod = R_align @ R.from_euler('z', -26.35, degrees=True).as_matrix() \
              @ R.from_euler('x',   0.8, degrees=True).as_matrix()

# Convert the combined rotation to URDF/ROS RPY: R = Rz(yaw) * Ry(pitch) * Rx(roll)
# (extrinsic/fixed-axis about the PARENT frame -- this is a different
# convention from how `prod` above was built, and that's fine: `prod` is
# just a 3x3 orientation; any valid Euler decomposition of it is equally
# correct, including this fixed-axis-about-parent one.)
pitch = np.arcsin(-prod[2, 0])
roll  = np.arctan2(prod[2, 1], prod[2, 2])
yaw   = np.arctan2(prod[1, 0], prod[0, 0])
rpy_deg = np.degrees([roll, pitch, yaw])   # -> [90.8, 26.35, 90.0] for this example
```

**Gotchas:**
- The most common pitfall: assuming URDF `rpy` components correspond one-to-one with protractor readings taken about the camera/object's own axes at the moment of physical measurement. They don't, unless those camera-axis measurements are first converted into parent-frame-referenced (extrinsic) RPY — a body-axis tilt and a parent-axis tilt are numerically different numbers describing the same real-world orientation.
- A rotation value's physical meaning depends on *which axis, in which frame, at what point in the composition* it was applied about — e.g. in one worked example, a 26.35° number was `Rz(-26.35°)` about the camera housing's own "up" axis, which (after an initial 90°-ish mounting alignment) corresponded to a depression angle from horizontal in the parent frame — not a rotation about the optical axis and not a rotation about the parent's Y axis, despite superficially "being the pitch-like number."
- Sensor/robotics frame conventions compound this: a camera's body/`_link` frame (commonly +X out the lens, +Y left, +Z up) and its `_optical_frame` (commonly +X right, +Y down, +Z out the lens) are two different axis conventions for the *same physical sensor* — don't interpret a rotation number without first confirming which of the two frames it's expressed in.
- A practical way to avoid the whole ordering/convention ambiguity when physically tuning an orientation: perform adjustments one at a time, record them **in the order actually performed**, and compose them as **intrinsic** (body-fixed) rotations — then the resulting product reads left-to-right exactly in the order the physical adjustments were made, with no need to mentally convert anything.

**Related:** [[camera-pose-error-sources]]

**Still unclear:**
This is standard rotation-composition math (consistent with the conjugation-identity derivation and verified numerically in the source discussion), not project-specific beyond the worked camera-extrinsic example. The worked example's specific numeric mounting values (`2.15 0.31 1.89`, `26.35°`, `0.8°`) are one project's particular camera joint, used here only to illustrate the general method — not universal constants. A portion of the step-by-step algebra tracing exactly which physical axis the 26.35° and 0.8° values correspond to under the parent frame was not fully captured verbatim; the stated conclusions are consistent on both sides of that gap but the intermediate steps weren't independently re-derived here.
