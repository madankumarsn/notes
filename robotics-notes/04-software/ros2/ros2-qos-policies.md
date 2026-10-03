---
topic: ROS2 QoS policies — reliability, durability, history
status: draft
layer: software
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# ROS2 QoS: reliability, durability, history

**One-line idea:**
ROS2 QoS has three independent knobs — **reliability** (does delivery retry while connected?), **durability** (do late-joining subscribers get old messages?), and **history/depth** (how many messages are buffered?) — publisher and subscriber settings must be *compatible* (not just "set correctly" on one side) or no connection is made at all.

**Why it exists:**
These three policies are easy to conflate (especially durability vs. reliability, which sound similar but answer completely different questions), and getting them wrong produces confusing failure modes — e.g. a subscription that silently never connects, or a late-starting node that never sees essential static data.

**Math / mechanics:**

- **Reliability — "if I'm connected now, do I get every message?"**
  - `RELIABLE`: don't drop messages; the publisher retries (ACK/NACK-based) until delivered to currently-connected subscribers. Use for data you must not miss (static config, commands).
  - `BEST_EFFORT`: send once; if lost, move on — no retransmits. Lower latency/overhead. Use for high-rate sensor streams where newer data supersedes older data anyway (camera, lidar).
  - **Compatibility is directional**: a subscriber requesting `RELIABLE` cannot connect to a `BEST_EFFORT` publisher (the publisher won't retry); a subscriber requesting `BEST_EFFORT` *can* connect to a `RELIABLE` publisher (it's just accepting less than what's offered). `RELIABLE`↔`RELIABLE` and `BEST_EFFORT`↔`BEST_EFFORT` always match.

- **Durability — "do late-joining subscribers get old data?"** This is a *publisher-side* cache feature, separate from reliability.
  - `VOLATILE` (default): publisher keeps no history for late joiners. Subscribe at t=30 → you only see messages published after t=30; anything earlier is gone forever. Right choice for live streams where only "now" matters.
  - `TRANSIENT_LOCAL` ("latched" in ROS1 terms): the publisher caches its own sent messages (up to its own history depth) and *replays* them to a subscriber that connects late. Right choice for config-like, published-once-or-rarely data (e.g. static transforms).
  - **"Last one" ≠ "last publisher on the topic."** Durability is per-publisher: if a topic has multiple `TRANSIENT_LOCAL` publishers, a late subscriber gets cached data from *each* of them (each up to its own depth), not just whichever happened to publish most recently topic-wide.
  - `RELIABLE` and `VOLATILE` (or `TRANSIENT_LOCAL`) are independent and combine freely — e.g. `RELIABLE + VOLATILE` means "no drops while connected, but nothing is remembered for late joiners."

- **History/depth — "how many messages are buffered, and what happens when the buffer is full?"**
  - `KEEP_LAST` (the common choice): keep only the N most recent messages (`N = depth`); a sliding window. When a new message arrives and the queue is full, the oldest is dropped.
  - `KEEP_ALL`: never drop; buffer everything until consumed (bounded only by resource limits) — rare in robotics, risks unbounded memory growth if a subscriber falls behind a fast publisher.
  - `depth` on a *publisher* with `TRANSIENT_LOCAL` durability means "how much history I cache for late joiners"; `depth` on a *subscriber* means "how large my receive queue is." These are conceptually different things that happen to share a name/number.

**Code:**
```python
# Typical pairing for a static/config-like topic (e.g. /tf_static)
rclpy.qos.QoSProfile(
    depth=100,
    durability=rclpy.qos.DurabilityPolicy.TRANSIENT_LOCAL,  # late joiners get cached data
    reliability=rclpy.qos.ReliabilityPolicy.RELIABLE,        # no drops
    history=rclpy.qos.HistoryPolicy.KEEP_LAST,               # sliding window of size `depth`
)

# Typical pairing for a live high-rate sensor stream
rclpy.qos.QoSProfile(
    depth=10,
    reliability=rclpy.qos.ReliabilityPolicy.RELIABLE,   # or BEST_EFFORT, depending on the driver
    history=rclpy.qos.HistoryPolicy.KEEP_LAST,
    # durability defaults to VOLATILE — no history for late joiners
)
```

**Gotchas:**
- Durability and reliability are **not** the same axis despite sounding related: durability = "do late joiners get *old* data," reliability = "while I'm connected, do I get *every* message." You can have any combination of the two.
- A `RELIABLE` subscriber silently fails to connect to a `BEST_EFFORT` publisher — there's no error thrown pointing at QoS by name; it just looks like the subscription never receives anything. If a subscription mysteriously gets zero messages despite the publisher clearly running, check QoS compatibility before anything else.
- For a topic with multiple `TRANSIENT_LOCAL` publishers (a common pattern for `/tf_static`, where e.g. a robot-state publisher and a camera driver both publish static transforms), a late-starting subscriber gets the accumulated cached data from *all* of them, not a single "winner" — this is exactly how a node can collect a full static TF tree despite starting well after all the publishers that built it.
- `ros2 topic hz`/basic debugging tools won't tell you *why* a subscription isn't receiving anything if the root cause is a QoS mismatch — you have to check the actual profiles on both ends.

**Related:** [[ros2-tf2-buffer-fundamentals]], [[ros2-executors-and-callbacks]]

**Still unclear:**
General ROS2 QoS semantics (consistent with the client library's documented behavior); the specific `/tf_static` RELIABLE+TRANSIENT_LOCAL+depth=100 profile and the contrasting camera-stream profile are from one project's code, used here as concrete, representative examples of the general pattern.
