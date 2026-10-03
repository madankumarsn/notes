---
topic: When product-instance TF actually gets published, and why RViz/bag replay can hide it
status: draft
layer: perception
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# Product-instance TF publishing conditions and RViz visibility

**One-line idea:**
A product-mapping node publishing per-detection TFs is conditional (only on success, and only outside a "tuning" mode), one-shot (dynamic `/tf`, sent once at the source image's timestamp, never republished), and — because it's one-shot — easy to mistake for "not working" when the real cause is RViz's own TF frame-timeout, a missing intermediate static frame, or a bag replay that never captured that one-shot message in the first place.

**Why it exists:**
A reusable checklist for "why don't I see my one-shot detection TF in RViz," which is a recurring question any time a perception node publishes a result as a dynamic transform rather than carrying it in a returned message.

**Math / mechanics:**
A one-shot detection TF goes through the exact same dynamic `/tf` broadcast mechanism as any continuously-republished transform, and is subject to the same receiving-side buffer retention window (see [[ros2-tf2-buffer-fundamentals]]) — there is no separate "one-shot" mechanism on the receiving end. The "one-shot" behavior comes entirely from the *publisher* calling `sendTransform()` exactly once per successful detection, rather than on a timer; once that single message ages out of a consumer's buffer (or past a visualization tool's own timeout), there is nothing left to look up until another detection is published.

**Code:**
```python
# Publishing is gated on BOTH success and not being in tuning mode
if result.success:
    product_tfs: list[TransformStamped] = []
    for product_instance in processor_result.product_instance_list:
        product_tf = self._make_product_tf(
            rgb_msg.header.stamp,                 # stamped at the SOURCE image time, not "now"
            product_instance.parent_frame,        # e.g. a pallet frame
            product_instance.child_frame,         # e.g. "{product_id}_{index:03d}"
            product_instance.transform,
        )
        product_tfs.append(product_tf)
    if not self.tuning_mode:
        self.tf_broadcaster.sendTransform(product_tfs)   # dynamic /tf, one-shot
```

```bash
# Verifying a one-shot TF is actually arriving, live
ros2 topic echo /tf --once
ros2 run tf2_ros tf2_echo <parent_frame> <child_frame>

ros2 action send_goal /run_product_mapping pika_ros_msgs/action/ProductMapping \
  '{requested_product_id: <id>}' --feedback
```

**Gotchas:**
- **Publishing is gated on two independent conditions**: the mapping action must report `success == True`, AND the node must not be in `tuning_mode` (used for hyperparameter tuning runs) — in `tuning_mode:=true`, TF is deliberately *not* broadcast, even on success; only other result messages are. A silent "no TF, but the action reported success" is expected behavior in that mode, not a bug.
- **All returned product instances get a TF, regardless of their `valid` flag.** Out-of-tolerance/out-of-ROI instances are still broadcast (marked invalid in the data, but still present as a transform) because downstream consumers (e.g. a depalletizing strategy) use invalid instances as occluders — don't assume "it has a TF" implies "it's a usable detection."
- **It's a one-shot dynamic `/tf` message, not `/tf_static` and not periodically republished.** The transform exists on the TF graph only for as long as the receiving buffer's retention window holds it (commonly ~10 s by default — see [[ros2-tf2-buffer-fundamentals]]) after that single publish. A downstream consumer that looks it up later gets a lookup failure that looks like a pose/accuracy problem but is actually just expiry.
- **RViz's own TF display has a "Frame Timeout" setting** (independent of the TF buffer's `cache_time`): if a one-shot transform is older than that timeout, RViz stops drawing it even if a lookup against it would still succeed. A detection "disappearing after about a second" in RViz, while the underlying pipeline otherwise works fine, is commonly this setting (default often ~1000 ms) rather than a data problem — increasing it (e.g. to several seconds or more) is a legitimate fix purely for eyeballing results, with no effect on the actual pipeline.
- **A missing intermediate static frame hides a perfectly good detection.** If the detection's parent frame (e.g. a pallet frame) is itself a child of some other static-TF publisher that isn't running in a given launch configuration, RViz may not render the detection at all when its Fixed Frame is set further up the tree (e.g. `map`) — even though the detection's own TF, and the pallet-to-detection transform, are both fine. Check `ros2 run tf2_tools view_frames` or `tf2_echo` on the intermediate link before suspecting the detector.
- **Pure `ros2 bag play` replay of an old recording will not show product-instance TF**, if that TF was only ever a live, one-shot `/tf` broadcast and wasn't itself captured into the bag at the time of recording (a common pattern: the bag only captures raw sensor/camera-TF data, while structured detection results are recorded separately, e.g. to a dedicated results topic). Replaying the bag alone replays the inputs, not the one-shot output — getting the detection TF back requires either re-running the mapping node against the bag, or republishing it from whatever topic the structured result (not the TF) was actually recorded to.
- **A mapping action can abort or succeed-with-zero-instances silently** unless explicitly checked via `--feedback`/result inspection — common abort reasons include a TF lookup failure, missing `CameraInfo`, or no synced RGB-D pair yet (e.g. if an action goal fires before bag/camera data has actually started flowing in a replay setup). Absence of TF is sometimes just the downstream symptom of an upstream abort, not a TF-specific bug.

**Related:** [[ros2-tf2-buffer-fundamentals]], [[ros2-bag-corruption-recovery]], [[ros2-bag-replay-tmux-launch-nuances]], [[product-mapping-pallet-offset-param]]

**Still unclear:**
These are one project's (pika_perception) specific publishing/gating rules, used here as a concrete illustration of a general class of "one-shot TF visibility" gotchas that apply to any node publishing detection results as dynamic transforms. Whether "mapping on the go" (running the mapper while still moving) changes any of these publishing conditions wasn't captured in the source material.
