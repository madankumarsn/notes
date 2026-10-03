---
topic: LOAM-style lidar mapping front end — per-scan component chain during live recording
status: draft
layer: estimation
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# LOAM-style mapping front end: per-scan component chain

**One-line idea:**
A LOAM-family mapping front end processes each lidar scan through a fixed chain — IMU fusion for heading, dead-reckoning integration for a drifting-but-always-available pose, feature extraction, insertion into a sliding local map, scan-to-map matching to correct the pose, and a final pose-fusion stage — producing both a live pose estimate and the raw artifacts an offline back end later turns into a finished, georeferenced map.

**Why it exists:**
A reusable template for what "the mapping front end" means in a LOAM-style (lidar odometry and mapping) system — useful for understanding or reviewing any similar lidar-first SLAM front end, not just one specific implementation.

**Math / mechanics:**
Per-scan pipeline, in processing order:
1. **IMU fusion** — removes gyro bias, applies the IMU's known mounting rotation, runs a Madgwick-style filter on the primary IMU. The fused IMU orientation becomes the heading source for every downstream stage.
2. **Dead-reckoning integration** — integrates speed using that fused heading (optionally blended with visual odometry, if a camera is present) into a continuously-published pose estimate. This stage has **no lidar in it at all** and drifts without bound on its own — its role is to always have *some* current pose estimate available between lidar-corrected fixes, and to deskew the next scan (interpolating this pose queue to each point's own timestamp within one lidar sweep, since lidar points within a single spin are not captured at the same instant).
3. **Scan registration / feature extraction** — re-bins the raw lidar cloud into rings, drops a "blind box" region close to the sensor (self-occlusion / mounting structure), motion-compensates the sweep using the integrator's interpolated poses from step 2, then splits the cloud into: sharp corner features, flatter surface features, the full-resolution cloud, and a per-scan odometry prior.
4. **Local map insertion** — inserts the extracted corner/surface features into a sliding voxel map centered on the vehicle (corners and surfaces stored in separate structures), and hands feature stacks + nearby map cells + a pose prior to the scan-matching stage.
5. **Scan-to-map matching** — runs point-to-plane (and/or point-to-edge) scan-to-map optimization, producing a pose correction relative to session start. This corrected pose feeds back into both the local map (so the map itself is built in the corrected frame, not the raw dead-reckoned frame) and into the dead-reckoning integrator (so future deskewing/dead-reckoning starts from the corrected estimate rather than drifting further from it).
6. **Pose fusion** — carries dead reckoning forward from the latest lidar-corrected fix, publishing a fused pose at IMU rate (much faster than the lidar-correction rate) — this fused output is the system's live vehicle pose while mapping is running.
A separate GPS-handling stage never moves the pose estimate directly; it just timestamps/pairs GPS fixes against the fused pose and logs them, for a later offline back-end step to use for georeferencing and/or loop-closure-against-GPS.

**Code:**
N/A — this is an architecture/pipeline description, not a code excerpt.

**Gotchas:**
- Mapping conventionally **always starts pose estimation at the origin**, regardless of any real-world starting coordinates supplied. A "start position" input (if the system takes one) is typically only held for later georeferencing of the finished map — it is not fed into the live pose estimate, so don't expect the live pose topic to read out in real-world/absolute coordinates during recording.
- Sensor messages are commonly ignored entirely until the system's state machine actually reaches an active "recording" state — a sensor publishing data before that point does not mean it's being processed.
- The dead-reckoning integrator (step 2) and the lidar-corrected pose (step 5/6) are genuinely different pose streams with different roles: the integrator is always-available-but-drifting and exists to deskew/interpolate between lidar fixes; the lidar-corrected pose is intermittent-but-accurate and is what actually keeps long-term drift bounded. Don't conflate "my odometry topic" with "my corrected/mapped pose topic" — check which one a downstream consumer is actually subscribed to.
- Whether a finished, georeferenced map is produced automatically on stopping, or only the raw front-end artifacts remain (for a separate manual/offline back-end run), is typically a configurable choice — don't assume stopping a recording session always yields a finished map.

**Related:** [[ros2-tf2-buffer-fundamentals]], [[bumblebee-pipeline-architecture]]

**Still unclear:**
This is a high-density architectural summary reconstructed from reading one project's (Lynx) mapping-mode source and component READMEs/docs in a single session, not independently cross-checked against the actual running system's behavior. Component-internal details (exact Madgwick filter parameters, voxel map cell sizing, how multiple parallel scan-matcher instances get assigned work) were not captured beyond the high-level architecture above — this project happens to run several scan-matcher instances in parallel per scan, but the assignment/arbitration logic between them wasn't detailed in the source material.
