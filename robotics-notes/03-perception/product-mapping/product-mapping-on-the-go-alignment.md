---
topic: Blocking-wait alignment gate for mapping while a platform is still moving
status: draft
layer: perception
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# "Map on the go": blocking alignment wait before capture

**One-line idea:**
An "on the go" mapping action blocks in a polling wait loop until the platform's position (in a map/world frame) is aligned with the target pallet along one axis — or has just passed it within an allowed overshoot — and only then maps the *exact RGB-D frame that triggered that condition*, rather than whatever frame happens to be newest once the wait exits.

**Why it exists:**
A reusable pattern for "capture exactly at a trigger condition, not at loop-exit time" — a subtle but important distinction whenever a wait loop is polling toward some threshold and the captured data needs to correspond precisely to the triggering instant, not to whatever arrived during the next loop tick.

**Math / mechanics:**
- Alignment error computed in the map frame: `error_y = camera.y - pallet.y`.
- Wait loop polls at a fixed interval (e.g. every 20 ms) but only **evaluates** the alignment condition on receipt of a *new* RGB timestamp — i.e. it's gated on fresh sensor data arriving, not purely on wall-clock polling cadence.
- Trigger condition: `|error_y| <= some_threshold`, OR the platform has just passed the target (sign of `error_y` flipped) within `max_overshoot_m` of the threshold — allowing capture slightly after perfect alignment rather than requiring the platform to stop exactly on it.
- On trigger, the RGB-D pair associated with the **specific stamp that satisfied the condition** is the one passed into the mapping/processing call — not a newer frame that might have arrived in the time it takes to exit the loop and start processing.

**Code:**
No production code was quoted verbatim; this is reconstructed from a flowchart-plus-walkthrough of the control flow, not a pasted implementation.

**Gotchas:**
- **Outcome branches of this action are all distinct and worth distinguishing when debugging:** `succeed()` (aligned, and the triggering frame mapped successfully), `canceled()` (an external cancel arrived during the wait), `abort()` — which itself can mean several different things: running in a replay-bag mode where alignment can't be satisfied the normal way, an unknown product or pallet ID, an alignment timeout (never reached the threshold within some time budget), or a mapping failure on the triggering frame itself. A bare "it aborted" without checking which of these fired can send debugging in the wrong direction.
- The wait evaluates alignment only on a *new* RGB stamp, not on every poll tick — if RGB data stops arriving (e.g. a sync/timing issue upstream), the wait loop keeps polling but never re-evaluates the condition, which looks identical to "waiting for alignment" even though the real blocker is missing sensor data.
- Capturing "the triggering frame" rather than "the newest frame once the loop exits" matters specifically because processing (segmentation, mapping) itself takes non-trivial time — by the time a newest-frame capture would run, the platform may have moved meaningfully further, defeating the purpose of a tight alignment gate.

**Related:** [[nms-by-prior-box-dims]], [[product-mapping-tf-publishing]]

**Still unclear:**
This control flow was reconstructed from a flowchart-building session on one specific node function (`_on_the_go_execute_cb` / its alignment-wait helper), not independently re-verified against the production source in this pass — treat the general "capture the triggering frame, not the loop-exit frame" pattern as the durable takeaway, and the specific polling interval/threshold values as one project's current tuning, not universal constants.
