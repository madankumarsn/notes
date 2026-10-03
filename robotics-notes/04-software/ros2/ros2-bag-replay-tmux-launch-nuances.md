---
topic: ROS2 bag replay nuances — sim time, launch-arg syntax, tmuxp multiline gotchas
status: draft
layer: software
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# ROS2 bag replay nuances (sim time, launch args, tmuxp)

**One-line idea:**
Replaying a ROS2 bag through a multi-pane tmux launch setup has several independent footguns — manual pane ordering, `use_sim_time`/`--clock` propagation, `ros2 launch` vs `ros2 run` argument syntax, and a YAML multiline quirk in tmuxp — any one of which silently breaks visualization or detection even though the underlying pipeline logic is fine.

**Why it exists:**
Debugging "why don't I see X when I replay this bag" usually isn't a detection/algorithm bug — it's one of a small, recurring set of plumbing issues around clocks, launch tooling, and terminal automation.

**Math / mechanics:**

- **TF tree gap vs. detection bug.** If a node publishes detections relative to a local frame (e.g. a pallet) rather than a global one (e.g. `map`), viewing them succeeds when RViz's Fixed Frame is the local frame, but fails when Fixed Frame is the global one — not because detection failed, but because the global→local static transform comes from a *different* node (e.g. a warehouse/scene description publisher) that the bag-replay launch setup didn't start. Always check "is the intermediate frame actually in the TF tree" before suspecting the detector.
- **Two different "now"s must agree for live visualization of bag data.** A node can process bag data correctly using the message's own `header.stamp` for internal lookups (so its math is unaffected), while RViz still fails to *display* the results, because RViz shows transforms relative to the *current* clock and treats old-looking stamps as stale. Fixing this requires every participant — the processing node, the bag player, any static-TF publisher, and RViz itself — to share the same simulated clock (`use_sim_time:=true` everywhere + `ros2 bag play --clock`), not just the node doing the math.

**Code:**
```bash
# Full set of panes that must all share sim time for bag-replay + RViz to agree:

# 1) the processing node
ros2 launch my_pkg my_pipeline.launch.py ... use_sim_time:=true

# 2) any static/scene TF publisher the pipeline depends on
ros2 launch my_pkg scene_description_publisher.launch.py ... use_sim_time:=true

# 3) RViz — note: ros2 run takes parameters via --ros-args -p, NOT launch-arg syntax
ros2 run rviz2 rviz2 -d my_view.rviz --ros-args -p use_sim_time:=true

# 4) the bag itself must publish /clock
ros2 bag play ~/bags/my_bag --clock
```

```bash
# Sanity checks when something isn't showing up
ros2 run tf2_ros tf2_echo map some_intermediate_frame   # does the link exist at all?
ros2 run tf2_ros tf2_echo map some_detection_frame      # full chain reachable?
ros2 bag info ~/bags/my_bag                             # which topics/frames does the bag actually contain?
```

**Gotchas:**
- **`ros2 launch` vs `ros2 run` take parameters differently.** `ros2 run pkg exe --ros-args -p name:=value` works for a plain executable. A *launch file* does not accept `--ros-args -p` at all — parameters like `use_sim_time` only work there if the launch file explicitly declares them as a `DeclareLaunchArgument` and passes `name:=value` on the `ros2 launch` command line. If a launch file doesn't already expose a parameter, passing it any way will fail or be silently ignored — you have to add the `DeclareLaunchArgument` yourself.
- **`--remap` is not `-p`.** `--remap name:=value` renames a *topic*; it is not how you set a parameter. Using `--remap use_sim_time:=true` gets parsed as a (deprecated-syntax) remap rule, which the target node then rejects as an unknown option — a confusing failure that looks unrelated to the actual mistake.
- **A space after a line-continuation backslash silently breaks multi-line shell commands.** `command \ \n next-arg` (note the space before the newline) does **not** continue the line in bash — the shell sees the backslash-escaped space as "nothing to continue," and the next line runs separately (or the first line errors on its own). This is easy to introduce when copy-pasting commands that have been reformatted/indented, and produces a bare "unrecognized arguments" error that looks like an API mistake rather than a shell syntax one.
- **Panes configured with `enter: false` in a tmuxp YAML can still auto-execute if the command string contains an embedded newline.** A YAML *folded block scalar* (`cmd: >`) joins lines with spaces but **keeps a trailing newline** at the end of the resulting string; tmuxp types that string into the pane verbatim, and the trailing `\n` is sent as an actual Enter keypress — overriding the intended "wait for me to press enter" behavior. The fix is `cmd: >-` (strip-trailing-newline block scalar) instead of `cmd: >`. Single-line `cmd: "..."` strings have no embedded newline and are unaffected — this only bites multi-line folded commands.
- **Manual pane ordering matters and isn't enforced by the tooling.** A pipeline that processes live-streamed topics (as opposed to having a bag baked directly into its input) needs the bag's data already flowing (camera topics, `/tf`) *before* you trigger the processing step — if panes are set to `enter: false` specifically so a human presses Enter at the right moment, triggering out of order produces a transient-looking failure ("missing message," "no synced pair yet") that can be mistaken for a timing/sync bug in the pipeline itself, rather than just doing the steps in the wrong order.

**Related:** [[camera-pose-error-sources]]

**Still unclear:**
These are tooling-specific behaviors (this project's `tmuxp`-based launch convention, this project's particular launch files) observed while debugging one bag-replay setup — the YAML folded-scalar trailing-newline behavior and the `ros2 launch`/`ros2 run` parameter-passing distinction are general (YAML spec / ROS2 launch system behavior respectively), but the specific pane layouts and launch files referenced are project-specific.
