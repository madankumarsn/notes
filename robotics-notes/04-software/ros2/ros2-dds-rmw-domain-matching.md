---
topic: ROS2 DDS/RMW implementation and domain matching — when it matters and when it doesn't
status: draft
layer: software
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# ROS2 DDS/RMW implementation and domain matching

**One-line idea:**
`RMW_IMPLEMENTATION` (which DDS vendor backs ROS2's middleware, e.g. CycloneDDS vs Fast-DDS) and `ROS_DOMAIN_ID` must match across every participant for two nodes to actually discover and talk to each other at runtime — but this is purely a *discovery-time* concern, irrelevant while only building or compiling code.

**Why it exists:**
It's easy to assume any ROS2-related environment variable might affect a build, when in fact DDS/RMW settings only matter once nodes are actually running and trying to find each other on the graph — a useful distinction for not chasing the wrong thing while debugging a build failure.

**Math / mechanics:**

- **`rcl`/`rclpy`/`rclcpp` sit on top of `rmw`, an abstraction layer over whichever DDS implementation is linked in** (commonly CycloneDDS `rmw_cyclonedds_cpp` or Fast-DDS `rmw_fastrtps_cpp`). Two nodes can only discover each other if they're using **compatible** RMW implementations and the **same** `ROS_DOMAIN_ID` (DDS's notion of an isolated network partition — nodes on different domain IDs simply never see each other, by design, even on the same machine/network).
- This only matters **at runtime**, when some other ROS2 process external to your own stack needs to see your nodes' topics (or vice versa) — e.g. interoperating with a stock ROS2 install that defaults to a different RMW than your project's container.
- A container run with `--net host` shares the host's network stack directly, so ROS2 discovery (typically UDP multicast) behaves the same as if the node were running natively on the host — no extra container-networking config is needed for DDS discovery to work, as long as RMW/domain settings agree.

**Code:**
```bash
# Per-machine overrides belong in a git-ignored local env file, not inline on the CLI
cp .env.local.example .env.local
# uncomment only if interoperating with a stock ROS2 install that defaults to Fast-DDS:
export RMW_IMPLEMENTATION="${RMW_IMPLEMENTATION:-rmw_fastrtps_cpp}"
```

**Gotchas:**
- Setting `RMW_IMPLEMENTATION=...` inline on the shell before invoking a script that runs the actual work **inside a Docker container** does not get forwarded into the container automatically — it has to go into a file that's actually sourced/mounted into the container (e.g. a `.env.local` the container reads), not just exported in the host shell that launches the script.
- A build/compile step (e.g. `colcon build` inside a container) does not care about `RMW_IMPLEMENTATION`/`ROS_DOMAIN_ID` at all — if a build is failing, these variables are not a productive thing to check; the failure is almost certainly a toolchain/permissions/dependency issue instead.
- A `--docker` flag on a wrapper script that already means "run this step inside the container" should not be combined with manually entering a container shell first — passing `--docker` again from inside the container tries to invoke Docker from inside Docker, which is not what's intended. Check what a wrapper script's `--docker`/similar flag actually means before combining it with "I'm already in the container" manually.

**Related:** [[ros2-client-library-stack]], [[ros2-qos-policies]]

**Still unclear:**
The RMW/domain-matching mechanics are general ROS2 behavior; the specific `.env.local` pattern and `--docker` wrapper-script convention are this project's (Lynx) build tooling, not a ROS2-wide standard — other projects may structure their Docker build wrapper differently.
