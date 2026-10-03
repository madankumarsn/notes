---
topic: TF two-time lookups through map/odom, and why localization corrections look like a "jump"
status: draft
layer: estimation
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# TF map/odom bridging and two-time lookups

**One-line idea:**
In a standard `map → odom → base_link` TF chain, `odom` is smooth/continuous (wheel/IMU-derived) while `map → odom` is the piece that jumps whenever localization corrects itself — so when tf2's `lookupTransform` "bridges" two different timestamps through a common frame, it's really doing two separate point-in-time lookups through that frame, not claiming the frame itself moved between those times.

**Why it exists:**
A reusable way to reason about "why did my detected object's position shift" on a localized mobile platform — the answer is almost always in which frame and which timestamp(s) the pose chain actually used, not a bug in the detector.

**Math / mechanics:**
Standard frame chain: `map --(localization, can jump)--> odom --(wheels/IMU, smooth)--> base_link --> cameras`. Pallet/landmark frames (e.g. a known pallet pose) are children of `map`, defined from an external database/CAD, not from camera measurements.

Pose written at mapping time (one single timestamp, no bridging needed):
\[
T_{\text{map}\leftarrow\text{product}} = T_{\text{map}\leftarrow\text{pallet}}^{\text{DB}} \; T_{\text{pallet}\leftarrow\text{product}}
\]
This single transform carries two independent, baked-in error sources: localization error at the moment the mapping photo was taken (`map ← base ← camera`), and any mismatch between the database/CAD pallet origin and the real pallet.

At a later detection/lookup time, if the code needs `target_frame ← product` and only has `product`'s pose in `map`, and if `product`'s own TF is no longer available at the *current* timestamp (it was last published back at mapping time), a **two-time bridge** is used — tf2's `lookupTransform(target, source, time, fixed_frame)` form:
\[
T_{\text{target}\leftarrow\text{product}} = T_{\text{target}\leftarrow\text{fixed}}(t_{\text{now}}) \; T_{\text{fixed}\leftarrow\text{map}}(t_{\text{now}}) \; T_{\text{map}\leftarrow\text{product}}
\]
(using `odom` as the fixed frame is a common choice: it appears effectively twice across the two queried timestamps in the general form, because the bridge looks up the "now" side and whatever's available of the "then" side both referenced through a frame treated as momentarily fixed — the exact composition depends on what's actually available at each timestamp.)

**Code:**
```text
# The error-attribution question this framework answers:
# "Why is the detected product offset from where it was mapped?"
#
# -> Check: is the SAME static/map-frame product pose being reused, or is a
#    fresh TF lookup happening through map -> odom at a DIFFERENT, later
#    "now" timestamp than when the product pose was originally written?
#
# If a static/republished map<-pallet<-product transform stayed valid at the
# exact query timestamp needed, a SINGLE-time lookup through map would
# suffice -- the two-time bridge exists only as a workaround for the product
# TF no longer existing at the query time, not because the math requires it.
```

**Gotchas:**
- **The frame that "jumps" is `map → odom` (localization correction), not `odom` itself.** `odom` by construction integrates smoothly (wheel odometry/IMU), it never jumps on its own — any apparent jump in a downstream (e.g. `base_link` or camera) pose traces back to a localization correction being applied to the `map → odom` link, not to odometry misbehaving.
- **A two-time TF bridge is a workaround for missing data at the query time, not a feature to rely on by default.** If a transform is only published once (e.g. at detection/mapping time) and later needs to be queried against a "now" that's long past, bridging through a fixed frame mixes an old, odometry-relative pose with a brand-new localization correction — which reintroduces localization error into what was otherwise a static, map-frame-anchored fact. Keeping the needed transform valid and queryable at the *exact* timestamp actually needed (via a single-time lookup) avoids this mixing entirely.
- **An offset observed at detection time is too easily over-attributed to "localization error."** Independent, separately-diagnosable error sources exist in the same pipeline: mapping-time depth/plane/instance-segmentation error, database/CAD pallet-pose mismatch vs. the real pallet, camera extrinsics (position/angle — see [[camera-pose-error-sources]]), arm forward-kinematics error (if the query frame is an end-effector), and viewpoint/timestamp differences between the mapping photo and the later detection. A post-hoc refinement step (e.g. ICP against a fresh local scan) run after an initial seed lookup can correct a localization/drift-based offset in that seed — so observing an offset in the seed pose alone doesn't mean the *final* pose used downstream is actually wrong.
- Changing which frame a pipeline treats as its "world frame" (e.g. `map` → `odom`) only changes which frame a *particular lookup* uses — it does not automatically make every other consumer of that pose (a product database anchored in `map`, a pallet frame defined in `map`) consistent with the new choice.

**Related:** [[ros2-tf2-buffer-fundamentals]], [[ros2-time-domains]], [[camera-pose-error-sources]]

**Still unclear:**
The exact tf2 two-time-bridge composition shown above (which transform gets looked up at which timestamp, and why `odom` specifically was chosen as the fixed frame in this project's code) is reconstructed from a conceptual discussion, not quoted verbatim from the actual lookup call site — the file/line of that specific `lookupTransform` call wasn't captured. Whether a design change to keep the product TF valid at the exact needed timestamp (avoiding the two-time bridge) was ever implemented is unknown; this was discussed as a proposal, not confirmed as shipped.
