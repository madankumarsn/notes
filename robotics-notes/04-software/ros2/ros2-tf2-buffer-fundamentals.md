---
topic: TF2 fundamentals — static vs dynamic transforms, tf_buffer, cache_time pruning
status: draft
layer: software
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# TF2 fundamentals: transforms, tf_buffer, and cache pruning

**One-line idea:**
`tf2` answers "where is frame B relative to frame A, at time T?" by maintaining a `tf_buffer` — an in-memory, time-indexed history of transforms (not a live publisher) — that keeps up to `cache_time` of history *per transform edge*, pruned by the timestamps on incoming transform messages, not by wall-clock time.

**Why it exists:**
A reusable model for debugging "TF lookup failed" / "extrapolation" errors, and for understanding why a node can be architected very differently in live operation vs. replaying from a recorded bag while reusing the exact same lookup code.

**Math / mechanics:**

- **A transform** is the pose (translation + rotation) of a child frame relative to a parent frame, valid at a specific time. Chains like `map → base_link → camera_link → optical_frame` are composed automatically at lookup time by walking the tree.
- **Two kinds of transform, two topics:**
  - *Static* (`/tf_static`): fixed forever (camera mounts, URDF joints). Published once or rarely.
  - *Dynamic* (`/tf`): changes continuously (robot moving, localization updating). Published at a regular rate.
- **`tf_buffer` (`tf2_ros.Buffer`) is a queryable history, not a live feed.** It stores incoming transforms indexed by `(parent_frame, child_frame, timestamp)`, and a lookup (`lookup_transform(target, source, time)`) walks the tree and interpolates between the two nearest stored samples at that time if needed.
- **`cache_time` (default 10s) is a per-edge retention window**, enforced on arrival: when a new transform arrives for a given parent→child pair at stamp `T_new`, entries for *that same edge* older than `T_new − cache_time` are dropped. Static transforms aren't pruned this way — once loaded, they effectively live forever.
- **Pruning uses the transform message's own stamp, not wall-clock "now."** This single fact is what makes a 10-second-default buffer work correctly for live operation (where stamps track real time) *and* for bag replay (where stamps are all from the recording's original timeline) — the buffer doesn't care what day it "really" is, only what day the incoming transforms claim to be from. See [[ros2-time-domains]] for the full wall-clock/ROS-clock/message-stamp distinction this depends on.
- **If a lookup asks for a time the buffer has no data near, you get an extrapolation error** — this can be a genuinely missing/sparse source (no `/tf` was published anywhere near that time) rather than a cache-size problem; increasing `cache_time` only helps when the *needed* history was pruned too early, not when the data was never there to begin with.

**Code:**
```python
# Setting up the buffer + listener (live mode — listener fills the buffer automatically)
self.tf_buffer = Buffer(cache_time=Duration(seconds=10.0))
self.tf_listener = TransformListener(self.tf_buffer, self)   # subscribes to /tf AND /tf_static internally

# A lookup at a specific moment, not "now"
t_query = Time.from_msg(rgb_msg.header.stamp)
tf_msg = self.tf_buffer.lookup_transform("map", camera_frame_id, t_query, timeout=...)
```

```text
# Replay mode — no live sensors, buffer filled manually from a recorded file:
Pass 1: read all /tf_static from the bag -> tf_buffer.set_transform_static(...)
        (optionally skip specific child frames and inject better calibration values instead)
Pass 2: read camera_info / synced rgb+depth / /tf chronologically
        -> tf_buffer.set_transform(...) for each /tf message
        -> after the scan, look up the chosen frame's TF at *that frame's own stamp*
           (same lookup_transform() call as live mode — only how the buffer got filled differs)
```

**Gotchas:**
- **`TransformListener` already subscribes to both `/tf` and `/tf_static` internally** to fill the buffer for lookups — you don't need your own subscription just to make `lookup_transform()` work.
- **A *separate*, explicit `/tf_static` subscription is still useful for a different purpose: recording.** `tf_buffer` is built for point-queries ("A relative to B at time T"), not for "give me every raw message that arrived on this topic" — there's no clean API to reconstruct the original batched `TFMessage`s from the buffer's merged internal tree. If you need to write a faithful `/tf_static` copy into an output bag (so it can be replayed later), accumulate the raw messages yourself in a side list rather than trying to extract them from the buffer.
- **Live-mode dynamic TF for a bag is usually captured differently from static TF.** Rather than subscribing to the full high-rate `/tf` stream for recording, it's common to just write the *lookup result* (one composed transform, at the moment of interest) as a single bag entry — you don't need 50-100 Hz of raw `/tf` history to replay one specific detection event later.
- **A short "live" cache size (e.g. the ROS-standard 10s default) can be entirely wrong for a long bag replay** if the code scans far ahead in time before doing its one lookup — but the fix isn't necessarily "make the cache huge"; a more targeted fix is to only load `/tf` within a window around the actual timestamp you'll query, which makes cache size stop mattering at all. A large cache size (e.g. 3600s) is a blunt, defensive workaround, not a precise fix.
- **A feature flag gating buffer size (e.g. "replay mode → big cache") can be wired to the wrong signal.** If "am I replaying" is determined by one launch parameter, but there's a *second*, independent way to trigger replay (e.g. a per-request path), using the bag via the second path without also setting the first parameter silently gives you the live-mode (small) cache — a subtle configuration trap worth checking explicitly rather than assuming "replay" is a single on/off switch.

**Related:** [[ros2-qos-policies]], [[ros2-time-domains]], [[camera-pose-error-sources]]

**Still unclear:**
General `tf2`/`tf2_ros` behavior (buffer semantics, pruning-by-stamp, static vs dynamic distinction) is consistent with ROS2 documentation; the specific two-pass bag-loading pattern, the dual static-TF-subscription-for-recording design, and the cache-size-gated-by-a-flag gotcha are drawn from one project's particular node implementation.
