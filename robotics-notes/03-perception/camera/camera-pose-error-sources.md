---
topic: Error sources in moving-platform camera-to-world pose estimation (extrinsics, timing, localization)
status: draft
layer: perception
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# Error sources in moving-platform camera-to-world pose estimation

**One-line idea:**
Projecting a camera detection into a world/map frame is \(p_{\text{map}} = T_{\text{map}\leftarrow\text{base}}(t) \cdot T_{\text{base}\leftarrow\text{cam}} \cdot p_{\text{cam}}\); extrinsics, the timestamp used for the lookup, and the live localization estimate each contribute an independent, roughly-additive error term, and a moving-vs-stationary comparison is the cheapest way to tell which one dominates.

**Why it exists:**
A reusable framework for "my detected object lands in the wrong place" on a moving robot: write the pose chain as a sequence of transforms, then ask which factor is actually wrong, rather than guessing.

**Math / mechanics:**

- **Compute latency vs. temporal misalignment are independent.** If the pipeline looks up \(T_{\text{map}\leftarrow\text{base}}\) at the **image's own `header.stamp`** rather than at wall-clock "now" when the detector finishes, then however long the detector itself takes to run (inference, plane-fitting, etc.) cannot bias the pose — it only affects throughput/latency, not accuracy. Confirming this stamp-anchoring property is the first thing to check before debugging anything else.
- **Extrinsic position error** → a constant offset, independent of range (camera is 1 cm further forward than believed → 1 cm error, always).
- **Extrinsic angle error acts as a lever arm:** \(e \approx d \cdot \tan\theta\) (small-angle: \(d \cdot \theta\)), where \(d\) is range to the target. Grows with distance, which is why it's usually the dominant term for a roughly-calibrated camera: 1° error gives ~1.75 cm at 1 m, ~3.5 cm at 2 m, ~7 cm at 4 m. This is also why eyeballing a camera's *angle* is much riskier than eyeballing its *position* — a degree is nearly invisible to the eye.
- **Timestamp offset** (the image's claimed capture time is off from the true exposure time by \(\Delta t\)) only matters while moving, and has two forms:
  - From translating: \(e = v \cdot \Delta t\) (0.5 m/s, 20 ms → 1 cm).
  - From rotating: \(e = d \cdot \omega \cdot \Delta t\) — a lever arm again, so it also scales with range (0.3 rad/s yaw, 2 m range, 20 ms → 1.2 cm).
- **RGB/depth sync skew:** if a synchronizer pairs color and depth frames within some tolerance, and depth gets projected using the TF looked up at the *color* frame's stamp, any actual skew between the two stamps times platform velocity is a misregistration error (e.g. 60 ms skew at 0.5 m/s ≈ 3 cm — comparable to a 2° extrinsic error). Measure the **actual** observed skew directly from the message stamps; it's usually much smaller than the configured tolerance, which is just the allowed *ceiling*, not the real value.
- **TF interpolation gap:** tf2 linearly interpolates between the two nearest localization samples. If localization publishes at, say, 20 Hz, the worst-case gap is ~50 ms, and linearly interpolating a curved path over that adds error proportional to how curved the path is during that window.
- **Localization wobble:** \(T_{\text{map}\leftarrow\text{base}}(t)\) is itself a noisy estimate (e.g. lidar-to-map matching), and unlike the other terms it is **not constant across runs** — it varies, which is what makes it the hardest of the three categories to pin down from a single trial.

**Code:**
```text
# Separating the three error categories — one experiment, no code changes needed:
Run detection on the same target:
  (a) once while the robot is moving
  (b) once while it is stationary
Compare the error in both.

  error ~equal in (a) and (b)      -> extrinsics (position or angle); timing work won't help
  error noticeably bigger in (a)   -> timestamp / sync-related
  error changes between repeats    -> localization noise (only this one is non-reproducible)
```

```text
# Three-step protocol to go further (requires a deterministic trigger first):
Step 1 — Deterministic trigger: pause playback at a fixed timestamp, or script the
         trigger off /clock, so every run processes the *exact same* frame.
Step 2 — Repeatability: trigger N times on that identical frame.
         Near-zero spread -> the error is systematic (e.g. calibration), not noise.
Step 3 — Viewpoint sensitivity: trigger on several consecutive frames as the robot
         approaches the same target (different viewpoints, same object).
         Position stays fixed in map/world coords -> geometry is self-consistent,
             any offset is a constant calibration bias.
         Position walks with viewpoint -> range-dependent error (extrinsic angle,
             or timing error while turning).
```

**Gotchas:**
- **The reprojection self-consistency trap.** If you compute a 3D point using camera X's own pixels + depth + extrinsics, then reproject that same point back into camera X's own image, the overlay looks perfect **even if the camera's extrinsics are completely wrong** — the transform error cancels against its own inverse. This is not evidence the extrinsics are good. Validate instead by viewing the result through a *different*, independently-trusted camera (e.g. a close-range end-effector camera) — that's the first view where the error doesn't cancel and becomes visible.
- **Human-in-the-loop triggering adds uncontrolled latency.** Watching a live view and pressing a button to trigger adds hundreds of ms (render lag + reaction time + round trip). At speed that's tens of centimeters of travel — you are not processing the frame you think you reacted to, and repeat "identical" runs will actually be on different frames. For repeatable measurement, trigger off a fixed bag/clock timestamp instead of by eye.
- **A pose broadcast once onto a short-lived dynamic TF buffer silently expires.** If a detection's pose is published as a one-shot dynamic transform rather than carried in a returned result/message (or republished, or sent as `/tf_static`), a downstream consumer that looks it up after the buffer's retention window (commonly ~10 s) gets a lookup failure — which has nothing to do with pose *accuracy* and is easy to misdiagnose as one. Carrying the pose directly in the detection's own result message sidesteps the expiry problem entirely (at the cost of the consumer needing to inject it into its own TF buffer at lookup time instead of doing a normal lookup).
- A detect-then-pick gap doesn't invalidate the detected pose if that pose is anchored to a static landmark (e.g. a pallet) rather than expressed relative to the moving robot — but the robot's *own* localization estimate keeps drifting during that gap, so the final grasp accuracy still degrades with elapsed time even though the stored detection itself is fine. A close-range re-detection right before the action (camera-to-base-link directly, bypassing the map) cancels this drift term entirely, because it never goes through localization.

**Related:** [[motion-blur-estimation]], [[rotation-composition-intrinsic-extrinsic]]

**Still unclear:**
This is from one real debugging session on one perception pipeline. The numeric examples (1° angle error, 20 ms timestamp offset, 60 ms sync tolerance, specific velocities/ranges) are the actual parameters discussed for that project's camera/platform, not universal constants — recompute for a different platform's speed, range, and publish rates. [[rotation-composition-intrinsic-extrinsic]] covers a distinct, related mechanics-of-euler-angles topic (how to interpret this same camera's URDF `rpy` extrinsics) — not merged in here since it's a different idea.
