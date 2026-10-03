---
topic: Estimating motion blur from a moving camera platform
status: draft
layer: perception
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# Estimating motion blur from a moving camera platform

**One-line idea:**
Motion blur from platform movement scales as \(\text{blur}_{px} = \dfrac{f_{px} \cdot v \cdot t_{\text{exp}}}{d}\); equivalently, the physical scene motion during one exposure is just \(v \cdot t_{\text{exp}}\), which comes out to a *fixed fraction* of a target's physical size regardless of range, because both the blur in pixels and the target's apparent size in pixels scale as \(1/d\).

**Why it exists:**
Before attributing a pose or segmentation problem to motion blur, a single back-of-envelope calculation can bound it and often rule it out — cheaper than running an experiment.

**Math / mechanics:**
- \(\text{blur}_{px} = f_{px} \cdot v \cdot t_{\text{exp}} / d\) — focal length (pixels) × speed × exposure time, divided by range.
- The scene itself translates by \(v \cdot t_{\text{exp}}\) during one exposure — a purely physical quantity, independent of the camera's focal length or the object's range.
- Because both \(\text{blur}_{px}\) and a target's apparent size in pixels scale as \(1/d\), their **ratio cancels range entirely**: \(\dfrac{\text{blur}}{\text{target width}} = \dfrac{v \cdot t_{\text{exp}}}{W_{\text{physical}}}\). This is the cleanest way to state a bound, since it holds across an entire operating range of distances at once.
- Worked example. Camera setup: 1280 × 800, 10 fps, 12.5 ms fixed exposure (auto-exposure off), \(f = 612.5\) px.

| Speed | 1 m | 1.5 m | 2 m |
|---|---|---|---|
| 0.25 m/s | 1.9 px | 1.3 px | 1.0 px |
| 0.5 m/s | 3.8 px | 2.6 px | 1.9 px |

  Worst case over the whole envelope is **3.8 px, at the closest range (1 m) and highest speed (0.5 m/s)**.

  Scene motion during one exposure: 3.1 mm at 0.25 m/s, 6.25 mm at 0.5 m/s. Against a ~400 mm case that is 0.8%–1.6% of the target's extent, independent of range, by the cancellation above.

  **Conclusion:** blur is a fixed small fraction of the target at any range, so it does not contribute meaningfully to the pose/mapping error.

**Code:**
```python
def blur_px(f_px: float, v_mps: float, t_exp_s: float, d_m: float) -> float:
    return f_px * v_mps * t_exp_s / d_m

def scene_motion_mm(v_mps: float, t_exp_s: float) -> float:
    return v_mps * t_exp_s * 1000

# e.g. f=612.5 px (from camera_info K[0]), t_exp=12.5 ms, auto-exposure off
for v in (0.25, 0.5):
    for d in (1.0, 2.0):
        print(v, d, blur_px(612.5, v, 0.0125, d))
```
```bash
# get real focal length rather than estimating it
ros2 topic echo /<camera>/color/camera_info --once   # K[0] = f_x (pixels)
```

**Gotchas:**
- **Fixed/manual exposure matters for the bound to hold.** With auto-exposure disabled, exposure time (and therefore blur) is deterministic and independent of scene lighting. If auto-exposure were on, any blur bound would be conditional on exposure actually staying near the assumed value, and would need restating.
- **"Blur is ~4 px against a 245 px target (under 2%)" is a defensible, falsifiable claim; "no motion blur" is not.** The latter invites someone to zoom in, find the small-but-nonzero blur that's actually there, and conclude the whole analysis was wrong.
- **State the operating envelope explicitly** ("at ≤0.5 m/s, 1–2 m range") rather than as an unconditional claim — blur scales linearly with speed, so a number computed at one speed silently stops applying if the platform later runs faster.
- **Back the calculation with a visual.** A 100%-crop of a sharp edge captured during motion (optionally placed next to the same edge from a stationary frame) is far more convincing in a write-up than the arithmetic alone — a near-identical pair of edges closes the question immediately.

**Related:** [[camera-pose-error-sources]]

**Still unclear:**
This is grounded in one specific camera/config (12.5 ms fixed exposure, 612.5 px focal length, 1280×800 at 10 fps). The formula and the range-cancellation argument are general optics, but the exact numeric table is particular to that project's camera and operating envelope.
