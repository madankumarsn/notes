---
topic: Bumblebee/Scorpion field-sensing stack — sensor pod, VIO+LOAM pipeline stages, log formats, and file formats
status: draft
layer: projects
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# Bumblebee: sensor pod, pipeline stages, log formats, PLY usage

**One-line idea:**
Bumblebee is a lidar+quadcam+IMU+triple-GPS handheld/drone sensor pod whose recorded logs are processed, offline, through a VIO → LOAM scan-matching → geo-referencing → semantic-labeling pipeline, run across three separate Docker containers because the pipeline's stages have mutually incompatible OS/ROS dependency requirements; `bumblebee-tools` is the processing-server half of this stack, and PLY is its intermediate working format for both point clouds and trajectories.

**Why it exists:**
A durable architecture reference for how a multi-sensor mapping pod's raw logs become a finished, georeferenced, semantically-labeled map — useful both for understanding this specific project and as a template for how VIO-seeded LOAM pipelines are generally structured end to end, including the often-overlooked step of making old vs. new log formats both feed the same downstream tooling.

**Math / mechanics:**
Sensor pod hardware: Ouster-class spinning lidar, a stereo "QuadCam" (own stereo calibration), an Xsens IMU, and **three** separate GPS receivers — three GPS units specifically so heading/attitude can be derived from GPS baselines, not just position from a single receiver.

Processing pipeline, in order:
1. **Log capture → ROS bag.** Two historical log formats exist, both normalized to the same downstream bag-based tooling:
   - **Phase 1** (older hardware/software generation): the pod writes a ROS1 `.bag` (lidar, lidar-IMU, triple-GPS) *and* a separate QuadCam directory (stereo images, Xsens IMU, calibration) at capture time, as two parallel outputs.
   - **Phase 2** (newer generation): the pod writes *only* a QuadCam-style directory (now including tarred lidar points/IMU/GPS plus metadata and calibration) — no bag exists on disk at capture time at all. A conversion tool (`qcToBag`-equivalent) synthesizes a ROS bag from this directory (mapping lidar IMU → an IMU topic, GPS → a GPS topic, lidar points → a points topic plus a second-return topic, visual-odometry pose tars → a VIO pose topic) purely so the same Phase-1-era downstream tooling keeps working unmodified.
   - A GPS HDOP-patching step exists specifically for Phase 2 logs, because HDOP was sometimes logged as zero there, which a visual-inertial odometry stage would otherwise treat as reason to reject that GPS fix.
2. **Visual-inertial odometry (VIO)** on the quadcam+IMU stream, with config knobs for IMU calibration time and a stationary-gyro-noise threshold, and an optional GPS-VIO factor graph that fuses GPS directly into the VIO solve. Emits pose through several stages (raw camera pose → locally-consistent → globally-consistent → final globally-consistent).
3. **LOAM scan matching**, seeded by the VIO pose (this is V-LOAM territory: vision seeding a lidar mapping pipeline) — refines pose and produces a trajectory file and a fused point cloud.
4. **Point cloud generation** from the refined trajectory + raw lidar.
5. **Geo-transform**: reprojects the cloud and trajectory into a GIS coordinate system (e.g. UTM).
6. **Geo closure**: a loop-closure pass against the GPS ground truth, correcting accumulated drift using the (sparser but globally consistent) GPS track.
7. **LAS conversion**: the final geospatial deliverable format, consumed by GIS tooling.
8. **Lidar-camera fusion / semantic labeling**: a trajectory interpolator and image interpolator project 3D points into camera images using an explicit, separately-calibrated `timestamp_offset` (lidar-camera clock offset); a calibration-check stage validates the lidar-camera extrinsic/timing calibration before any label projection is trusted; finally, image-space semantic labels are projected onto the point cloud's individual scans. Only after this geometric fusion stage does an ML semantic-segmentation model (e.g. a point-transformer architecture) run on the now-labeled or label-ready cloud.

**Code:**
```cpp
// utils/include/utils/point_types.h — the trajectory point type actually
// used by the pipeline's PLY trajectory files (not just x,y,z):
struct TrajPoint
{
  PCL_ADD_POINT4D
  float roll;
  float pitch;
  float yaw;
  double time;
};
```
```text
# PLY header, binary variant (used throughout the pipeline for both point
# clouds and trajectories — binary chosen over ASCII because a fused
# multi-channel-lidar map is far too large for a practical ASCII dump):
ply
format binary_little_endian 1.0
element vertex 1234567
property float x
property float y
property float z
property uchar red
property uchar green
property uchar blue
end_header
<raw vertex bytes...>
```

**Gotchas:**
- **Handheld motion is substantially harder for LOAM than vehicle-mounted motion.** No wheel odometry / vehicle motion model is available to constrain the solve, handheld rotation introduces worse motion distortion within each scan (so deskewing/interpolation quality matters more), and open/flat/low-structure environments carry a higher degeneracy risk (lidar scan-to-scan registration becomes ill-conditioned when there isn't enough 3D structure to constrain all six pose degrees of freedom).
- **Three separate Docker containers exist because of genuinely conflicting dependency requirements**, not organizational preference: the LOAM stage needs an old ROS1/Ubuntu-20.04-era toolchain, while the VIO stage needs a much newer Ubuntu release — there is no single container image that satisfies both without one of them being unsupported.
- **PLY only carries the fields actually declared in its own header** — readers ignore anything not declared, and the format imposes no semantic distinction between "this PLY is a point cloud" and "this PLY is a trajectory" beyond what fields happen to be present. The pipeline keeps point-cloud and trajectory PLYs in matched filename pairs (e.g. a `_transformed` suffix applied to both together) precisely because downstream stages (like colorization) expect that pairing and have no other way to know which trajectory belongs to which cloud.
- **PLY is a convenient research/tooling interchange format** (point-cloud libraries, the VIO/LOAM tools themselves, colorization, point-cloud viewers) **but is not a GIS standard** — the pipeline's actual geospatial deliverable is a LAS file, produced by a dedicated conversion step at the very end, specifically because GIS-facing tools expect LAS's embedded CRS/georeferencing metadata that PLY has no standard way to carry.
- **Phase 2's "no bag on disk" log format is a real architectural shift**, not just a renamed directory — any tooling/CLI that assumes a bag path exists on disk (e.g. a flag whose historical meaning was "path to an existing bag") needs to be reinterpreted for Phase 2 logs as "name of a bag to be *generated*," since the bag is synthesized from the QuadCam-style directory rather than being a pre-existing artifact.
- If map/registration quality is a hard upper bound on downstream semantic-classification quality (a label is only as good as the point/pose it's attached to), the estimation/mapping stage is squarely upstream of the whole ML labeling effort — a reason to prioritize mapping/registration correctness investigation before tuning the semantic model itself when results look wrong.

**Related:** [[lynx-mapping-frontend-pipeline]]

**Still unclear:**
Technical pipeline details were gathered from reading the actual `bumblebee-tools` repo (READMEs, pipeline scripts, a point-types header, an architecture diagram) in one session — accurate as of that read, but not independently re-verified here; treat exact config schema and file-layout specifics as approximate rather than quoted verbatim throughout. Exact numeric config defaults (IMU calibration time, stationary-gyro-std-dev threshold) weren't captured beyond their existence and role. A separate, non-technical portion of the source material (career-strategy discussion, including web-search-derived market/company figures) is intentionally excluded from this note as out of scope for a project-architecture reference.
