---
topic: ROS2 executors — spin is event-driven, callbacks are serialized by default
status: draft
layer: software
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# ROS2 executors: spin is event-driven, not periodic

**One-line idea:**
`rclpy.spin(node)` is an event loop that waits for work and runs callbacks as they arrive — there's no fixed rate — and with the default single-threaded executor, only one callback on that node runs at a time; anything else is queued, not interrupted or parallelized.

**Why it exists:**
"Continuous" (as in `spin` running "continuously") is easy to misread as "running on a tight loop" or "at some rate." Understanding spin as event-driven, and understanding what happens when events pile up, explains a whole class of "why did my callback run late / in what order" questions.

**Math / mechanics:**

- **`spin` has no fixed rate.** It blocks, waits for an event (message arrival, timer fire, service/action call), runs the matching callback, then goes back to waiting. How often callbacks run depends entirely on how often events occur — if a topic publishes at 10 Hz, you get ~10 callbacks/sec *while messages are arriving*; if nothing is happening, `spin` just blocks (not busy-polling).
- **Default executor is single-threaded: callbacks on a node are serialized.** If callback A is running when message 2 and message 3 arrive, they queue; `spin` does not interrupt A. When A finishes, the executor picks the next queued callback. This holds both for repeated messages on the same subscriber and for callbacks on *different* subscribers/timers on the same node — there's no concurrency by default.
- **If messages arrive faster than they're processed, they queue subject to QoS** (history policy + depth — see [[ros2-qos-policies]]). With `KEEP_LAST` and a small depth, older queued messages get dropped rather than processed, so a slow callback causes you to skip stale data rather than fall further and further behind.
- **A periodic timer (e.g. a 500 ms tick) and `spin`'s event-driven callbacks are two independent mechanisms that can run concurrently *if* the timer runs on its own thread** (e.g. a tree-tick library spinning up a background timer thread) separate from the thread calling `spin()`. In that case the periodic logic and the event-driven callbacks genuinely overlap in time — on different threads — and any state they both touch (e.g. a shared blackboard) needs to be safe to access from both contexts.
- **To get actual parallel callback execution**, you need a `MultiThreadedExecutor` plus callback groups (`ReentrantCallbackGroup` or `MutuallyExclusiveCallbackGroup`) — the default single-threaded `spin()` never does this on its own, and introducing it means taking on real thread-safety responsibility for any shared state the callbacks touch.

**Code:**
```python
# Two independent mechanisms running concurrently on different threads:
tree.tick_tock(period_ms=500.0)   # periodic: re-evaluate logic every 500 ms (own thread)
...
rclpy.spin(tree.node)             # event-driven: process ROS callbacks as they arrive (this thread)
```

```text
Message 1 arrives → callback A starts (takes 50 ms)
Message 2 arrives at +10 ms → queued, waits
Message 3 arrives at +20 ms → queued, waits
A finishes at +50 ms → executor runs the next queued callback (B), then C
```

**Gotchas:**
- Don't assume two subscriber callbacks on the same node ever run "at the same time" just because `spin()` looks like it's always active — with the default executor they are strictly serialized, one at a time, in arrival order.
- A periodic tick thread (timer-driven) and the `spin()` thread (event-driven) are a genuine exception to that serialization — they run on separate threads and *can* overlap. Any shared mutable state between "periodic logic" and "ROS callback logic" needs explicit thread-safety consideration, even though callbacks *within* `spin()` itself never overlap each other.
- A callback that takes a long time to run blocks everything else queued behind it on that same executor — a slow callback is a direct cause of lag on *every other* callback on that node, not just the topic it's attached to.

**Related:** [[ros2-client-library-stack]], [[ros2-qos-policies]]

**Still unclear:**
General ROS2 executor behavior; the periodic-tick-thread example is specific to a `py_trees_ros`-style behavior tree using `tick_tock()` alongside `rclpy.spin()`, but the underlying single-threaded-executor-serializes-callbacks behavior is general to any `rclpy.spin()` usage with the default executor.
