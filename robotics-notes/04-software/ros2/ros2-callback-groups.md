---
topic: ROS2 callback groups — mutually exclusive vs reentrant, and shared-state risk
status: draft
layer: software
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# ROS2 callback groups: mutually exclusive vs reentrant

**One-line idea:**
A callback group controls which of a node's callbacks are allowed to run concurrently under a `MultiThreadedExecutor`: `MutuallyExclusiveCallbackGroup` lets only one callback in the group run at a time, `ReentrantCallbackGroup` lets any number run concurrently — putting a long-blocking callback in its own group (separate from time-critical subscriptions) is the standard fix for "my subscriptions freeze while an action/service call is in progress," but a `ReentrantCallbackGroup` also means every callback sharing it needs to tolerate running concurrently with itself and with the others in the same group.

**Why it exists:**
Grouping is the main lever for controlling concurrency in `rclpy`/`rclcpp`, and the two group types answer opposite questions ("run one at a time" vs "run freely in parallel") — picking the wrong one either reintroduces the blocking problem you were trying to fix, or introduces real concurrency bugs on shared mutable state.

**Math / mechanics:**

- A subscription/timer/service/action server with **no explicit callback group** lands in the node's default group, which is mutually exclusive — i.e. by default, nothing on a node runs concurrently with anything else on that node (consistent with [[ros2-executors-and-callbacks]]'s single-threaded-executor default).
- A long-blocking callback (e.g. an action's execute callback that polls/waits for tens of seconds) occupies its group's "one callback at a time" slot for that whole duration. If camera/TF subscription callbacks share that same group, they're starved until the long callback finishes — the usual symptom is "my sensor data looks frozen/laggy exactly while some other operation is in progress."
- The standard fix is a `MultiThreadedExecutor` (so more than one callback thread exists at all) **plus** putting the long-blocking callback in its **own** group, separate from the time-critical subscription callbacks — this frees the subscriptions to keep running on a different thread while the long callback blocks on its own.
- Putting **multiple** things in one `ReentrantCallbackGroup` to solve "don't block each other" is a stronger, riskier choice than it looks: it doesn't just free them from blocking one another, it permits literal concurrent execution of all of them — including two copies of the *same* callback running at once if, e.g., two action goals of the same type are both accepted. Anything they all touch (shared node state, a shared processor object, a shared bag writer) needs genuine thread-safety, not just "it used to work serialized."

**Code:**
```python
# Separate the long-blocking action from time-critical subscriptions:
self._long_running_cb_group = MutuallyExclusiveCallbackGroup()   # only one at a time within itself
self.action_server = ActionServer(
    self, MyAction, "my_long_action",
    execute_callback=self.execute_cb,          # blocks for tens of seconds
    callback_group=self._long_running_cb_group,
)
# camera/TF subscriptions stay on the node's default group (or their own),
# and keep being serviced on a different executor thread while the action runs.
```

```python
# A single shared reentrant group across two action servers means goals of
# EITHER type (including two of the SAME type) can run fully concurrently:
self._action_cb_group = ReentrantCallbackGroup()
self.action_server_a = ActionServer(..., callback_group=self._action_cb_group)
self.action_server_b = ActionServer(..., callback_group=self._action_cb_group)
# -> only safe if both execute callbacks' shared state (e.g. self.processor,
#    a cached message buffer, a bag writer) can tolerate concurrent access.
```

**Gotchas:**
- Three distinct policies are worth naming explicitly when designing multiple action servers on one node: (A) one shared **exclusive** group for all of them (only one action instance runs at a time, of any type), (B) one exclusive group **per action type** (different action types can run concurrently, but never two goals of the *same* type), (C) one shared **reentrant** group (everything, including duplicate goal types, can run concurrently). (C) is the easiest to reach for by copy-pasting a working reentrant-group example, but is also the one most likely to silently violate shared-mutable-state assumptions unless that was specifically designed for.
- A goal callback that always `ACCEPT`s, combined with a shared reentrant group, means nothing in the code is actually preventing two mapping/processing goals from running concurrently and mutating the same cached state — if that's not intended, the fix belongs in the callback-group policy (or an explicit "reject if busy" check in the goal callback), not somewhere downstream.
- Before reaching for callback groups at all, check whether a long "blocking" wait can be restructured as a non-blocking state machine driven by the node's own event loop instead (timers/callbacks rather than `time.sleep()` polling) — that sidesteps the whole "which group" question, at the cost of more complex callback-based control flow.

**Related:** [[ros2-executors-and-callbacks]], [[ros2-client-library-stack]]

**Still unclear:**
General `rclpy`/`rclcpp` callback-group semantics are consistent with documented executor behavior. The specific three-policy framing (A/B/C above) and the "goal callback always ACCEPTs onto a shared reentrant group" risk are from one project's PR-review discussion about a specific pair of action servers (mapping, on-the-go mapping) that share mutable processor/buffer state — which policy the team ultimately chose wasn't confirmed in the source material.
