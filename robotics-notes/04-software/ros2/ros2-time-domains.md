---
topic: ROS2 time domains — wall clock vs ROS clock vs message stamp, live/sim/bag-replay
status: draft
layer: software
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# ROS2 time domains: wall clock, ROS clock, message stamp

**One-line idea:**
There are three distinct notions of time in ROS2 — the OS's wall clock, a node's `now()` (which is wall clock unless `use_sim_time` redirects it to the `/clock` topic), and a message's own `header.stamp` (set by whoever published it) — and they only coincide by convention in live operation; simulation and bag replay are precisely the cases where they diverge, and code that reasons using the wrong one breaks silently rather than loudly.

**Why it exists:**
A huge class of "it works live but not on a replayed bag" (or vice versa) bugs comes down to which of these three times a piece of code actually queries — this is the conceptual key to debugging all of them at once, rather than case-by-case.

**Math / mechanics:**

- **The three layers:**
  - **Wall clock** — real OS system time (what `date` shows). Always moves forward at 1 s/s, independent of ROS entirely. Used for logging, code-level timeouts, "how long did this take."
  - **ROS clock** (`node.get_clock().now()`) — what a node treats as "now." Governed by `use_sim_time`: `false` (default) → wall clock; `true` → whatever the `/clock` topic says.
  - **Message stamp** (`header.stamp`) — the time *inside* a specific message, set by its publisher, meaning "this data is valid as of this instant." Not automatically the same as either of the above, though publishers conventionally set it from `get_clock().now()` at publish time.
- **Live mode:** all three usually line up closely (within processing delay / sensor-clock jitter), because `use_sim_time=false` makes ROS clock = wall clock, and publishers stamp messages from `get_clock().now()`. Close, but not guaranteed identical on every message — a camera's hardware clock, publish-time processing delay, or a bug can all introduce a small gap.
- **Simulation (`use_sim_time=true`, a simulator publishing `/clock`):** one shared *fake* timeline. ROS clock follows `/clock` (which can run paused, slowed, or sped up relative to wall clock); every participating node's publishers stamp messages from that same ROS clock. So ROS-clock and message-stamps agree with each other — just not with the wall clock, which is irrelevant to the system.
- **Bag replay — two different mechanisms with very different implications:**
  - **A) `ros2 bag play --clock` + `use_sim_time:=true` on all participating nodes.** The bag player publishes synthesized `/clock` messages that step through the recording's original timeline. Every `use_sim_time`-enabled node's `now()` then follows that timeline — effectively, "ROS pretends it's the recording day again," system-wide. This is mechanically identical to simulation: one shared (now record-time) timeline that *all* nodes' clocks and new message stamps follow.
  - **B) Manually injecting bag data into a buffer without publishing `/clock`** (e.g. reading a file directly and calling `tf_buffer.set_transform(...)`). Here there is **no shared timeline** — the node's own `now()` stays on wall clock (today), while the *injected data* carries its original recording-time stamps. This only works correctly if every lookup explicitly queries using the data's own stamp (e.g. `lookup_transform(..., t_query=image.header.stamp)`) rather than ever asking "what does the buffer have for right now" — because "right now" (today) and the injected data's timeline (recording day) are simply different, disconnected time domains that the system never reconciles.
- **`/clock`'s values come from the bag player itself**, not from something embedded as a special topic in the bag (in the common case): the player walks the bag's recorded messages in time order and publishes a `Clock` message advancing through that same recorded timeline, pacing playback by `--rate`. Nodes with `use_sim_time:=true` simply subscribe to `/clock` and treat its value as "now" — no OS date is ever changed; ROS nodes just stop consulting it.
- **`rclpy.time.Time()` (no arguments) is literally zero** (`sec=0, nanosec=0`) — not "now." Two very different conventional uses of that zero value: as a *message stamp* on a static transform, it's a placeholder meaning "timeless / always valid" (static transforms don't have a meaningful "as of" instant); as a *query time* passed to `lookup_transform`, zero is a tf2 convention meaning "give me the latest available transform" — a different meaning in each context, and neither means "the current wall-clock instant."

**Code:**
```python
# Correct: query a buffer using the data's own timestamp, not "now"
t_query = Time.from_msg(rgb_msg.header.stamp)      # "when this photo was taken"
tf_msg = tf_buffer.lookup_transform("map", cam_frame, t_query)

# Would break on replay-without-/clock: asking the buffer for "today"
t_query = self.get_clock().now()                    # wall clock, e.g. "today" — wrong domain
tf_msg = tf_buffer.lookup_transform("map", cam_frame, t_query)   # -> lookup failure
```

```bash
# Mechanism A: whole-system shared replay timeline
ros2 bag play my_bag.mcap --clock                     # player publishes /clock from bag timeline
ros2 run my_pkg my_node --ros-args -p use_sim_time:=true   # node's now() follows /clock
```

**Gotchas:**
- **"It feels like one clock" in live mode is a coincidence of convention, not a guarantee.** Code that implicitly assumes "message stamp ≈ ROS now ≈ wall clock" will silently produce wrong results the moment it's run against a simulator or a hand-rolled bag-replay path — not because the code is "buggy" in the live case, but because it was relying on an assumption that only holds there.
- **Static transforms conventionally use a zero stamp** (`Time().to_msg()`), which is a deliberate "no specific time, always valid" convention — don't read a zero stamp on a static transform as a bug or as "epoch 1970."
- **Two independent pieces are both required for full-system sim-time replay**: a `/clock` publisher (the bag player, with `--clock`) *and* `use_sim_time:=true` on every node that should follow it. Either alone does nothing useful: `--clock` with no `use_sim_time` nodes is simply ignored; `use_sim_time:=true` with nobody publishing `/clock` leaves a node's `now()` frozen (typically at zero) until some `/clock` publisher appears.
- **A node that manually injects bag data into a buffer (mechanism B above) deliberately avoids needing `/clock` at all** — but that design choice only remains correct as long as *every* time-sensitive lookup in that code path explicitly uses the data's own stamp. A single accidental `self.get_clock().now()` slipped into that code path reintroduces the wall-clock-vs-recording-time mismatch bug.
- **TF buffer pruning (`cache_time`) is itself an instance of "uses message-stamp time, not wall clock"** — which is exactly why a buffer filled with recording-day stamps doesn't get instantly pruned to empty just because wall-clock "today" is weeks later. See [[ros2-tf2-buffer-fundamentals]] for the pruning mechanics this depends on.

**Related:** [[ros2-tf2-buffer-fundamentals]], [[ros2-bag-replay-tmux-launch-nuances]]

**Still unclear:**
General ROS2 time/clock semantics (`use_sim_time`, `/clock`, `rclpy.time.Time()` conventions) are consistent with documented ROS2 behavior. The "manual injection without `/clock`" pattern (mechanism B) is one specific project's bag-replay design choice, used here as a concrete illustration of a disconnected-timeline replay strategy — not every project's bag-replay code is structured this way.
