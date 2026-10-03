---
topic: Lidar multi-return (second-return) physical mechanism and intensity-value encoding
status: draft
layer: perception
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# Lidar multi-return: physical mechanism and intensity encoding

**One-line idea:**
A lidar "second return" is the *same* laser pulse producing two distinct echoes (e.g. partially blocked by a leaf, with the rest of the beam continuing to the ground behind it) — not a second physical sensor or a second scan — and when a pipeline needs to visualize both returns in one merged intensity column, encoding "this is a second return" as a large additive offset (well above the real sensor value range) is a simple, inspectable way to keep the two populations separable downstream.

**Why it exists:**
Multi-return lidar output is easy to misinterpret (as if it were two different sensors or two full separate sweeps), and intensity values that look unexpectedly large in a visualization are often an inspection-friendly encoding trick, not raw physical sensor noise.

**Math / mechanics:**
- **Range from time-of-flight:** \( \text{range} = \dfrac{\Delta t \cdot c}{2} \) (round-trip time, speed of light, divided by two for the round trip).
- **Second-return intensity encoding:** `displayed_intensity = sensor_value + (10000 if second_return else 0)`. The `10000` offset is an arbitrary flag value chosen to sit comfortably above the real sensor intensity range (observed up to roughly 5687 in one dataset) so that, once merged into a single intensity column, first-return and second-return populations never overlap and can be told apart just by thresholding the displayed value. The same trick is used on a separately 0–255-rescaled "reference" cloud with the same offset convention.
- **Scan geometry (example sensor, N-channel spinning lidar):** with N vertical channels firing ~simultaneously per column and M columns per full rotation, one rotation produces N×M points; at a given rotation rate, points/sec = N×M×rotations/sec. The M columns within one rotation are captured at different instants spread across the rotation period — this matters for motion compensation/deskewing, since points from the same "frame" are not actually simultaneous.
- Observed second-return rate in one dataset: roughly 23% of total points (indicating partial occlusion — e.g. vegetation canopy — was common in that environment).

**Code:**
```cpp
// A legacy-origin marker value, reused as the "this is a second return" tag
// added to intensity so merged output stays separable downstream.
static constexpr float TERTIARY_INTENSITY_TAG = 10000.0f;
```

**Gotchas:**
- **Second returns are not a second sensor and not a repeated scan.** They're the same emitted pulse producing multiple echoes because the beam was only partially blocked by a near object (e.g. a leaf), with the unblocked portion continuing on to reflect off something farther away (e.g. the ground). Treating "first return" and "second return" clouds as if they came from independent sensors is a category error that will misdirect debugging.
- **An unexpectedly large intensity value (e.g. > ~10,000) in a visualized/merged cloud is very likely an encoding tag, not a sensor anomaly** — check whether the pipeline adds an offset to mark second (or later) returns before assuming a hardware or calibration issue.
- **A config mode that stores something other than sensor intensity in the "intensity" field** (e.g. a small integer marking which processing/source path a point came from) can make a cloud's intensity column look like garbage (e.g. only values of "1" and "2" everywhere) when in fact that's working as designed — confirm which output mode a pipeline is configured for (sensor-intensity passthrough vs. a source/path marker) before interpreting the values as physical measurements.
- **Second returns can legitimately be included in a visual/map cloud while being excluded from the pose-solving/scan-matching path.** This is a deliberate choice — the extra points add density/visibility (e.g. seeing through canopy to the ground) without being allowed to perturb the pose estimate, which is exactly the tradeoff you'd want if second returns are systematically noisier or less geometrically reliable than first returns for registration purposes.
- Points captured at different instants across one rotation (not a single simultaneous "frame") is a standing reason deskewing/motion-compensation matters for any spinning lidar — see [[lynx-mapping-frontend-pipeline]] for how one scan-registration stage handles this via pose-queue interpolation.

**Related:** [[lynx-mapping-frontend-pipeline]]

**Still unclear:**
The specific `TERTIARY_INTENSITY_TAG = 10000.0f` constant, the exact channel/column counts, and the observed 23% second-return rate are one project's (Lynx, using an Ouster-class sensor) particular numbers, carried over from an older codebase's convention — useful as concrete, representative examples, not universal lidar constants. Exactly how the two output modes (`sensor` vs. a source/path-marker mode) are selected/configured wasn't fully captured beyond their existence and observed effect.
