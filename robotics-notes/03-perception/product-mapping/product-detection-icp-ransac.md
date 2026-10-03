---
topic: Edge-ICP pose detection with a rotation-flip gate, refined by RANSAC face-plane corner fitting
status: draft
layer: perception
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# Edge-ICP + RANSAC face-plane corner refinement for box pose detection

**One-line idea:**
A box's precise pose is found in two deliberately separate stages that each use the *right* data for the job: edge-ICP matches a template wireframe against the depth cloud's noisy edge/discontinuity points to get rough position (edges are the only features that constrain position along a flat face, but are also the noisiest data a depth sensor produces), then a RANSAC-refined plane-intersection stage uses the *clean* face data (not the noisy edges) to compute a precise corner as the intersection of up to three mutually-perpendicular fitted planes.

**Why it exists:**
A general pattern for precision pose refinement: use the minimal, noisy signal needed to get a correct *rough* pose and reject grossly wrong matches (ICP + a sanity gate), then re-derive the *precise* geometry from whichever subset of the data is actually clean, rather than trusting the noisy signal for both jobs.

**Math / mechanics:**
- **Why edges, despite being noisy:** a flat face alone gives no positional constraint along itself (a plane can slide along its own surface and still "fit"); the box's physical edges are what pin down position along the face. But depth cameras are noisiest exactly at edges/discontinuities, where the sensor straddles a near surface and a far surface in the same local neighborhood.
- **Rotation gate (hard reject):** `rotation_degrees` is the axis-angle magnitude of ICP's fitted rotation, measured **relative to the expected product pose prior** (not an absolute world orientation) — always in `[0°, 180°]`. If `rotation_degrees > expected_rotation_threshold_degrees`, the detection is rejected outright.
  - This is **not** specifically a "detect 90°/180° flip" check — it rejects *any* deviation from the prior above the threshold. Flips are simply the practically relevant motivating case: a rectangular face is exactly symmetric under 180° rotation about its own normal, and a near-square face is nearly symmetric under 90°, so ICP can converge to a flipped pose with an equally good fitness score — fitness only measures point-to-point distance, which is blind to orientation symmetry.
  - Flips matter downstream because a "use bottom pose" convention (if one exists) swaps which edge is treated as top vs. bottom under a 180° flip, which would misplace a downstream end-effector action.
  - Threshold is per-product-tunable: a default around 30° catches flips comfortably (well under both 90° and 180°) while still rejecting genuinely bad alignments; it can be tightened (e.g. 20–25°) for products needing stricter alignment, or loosened (e.g. up to ~65°) for products whose expected real-world misalignment tolerance is larger — a loosened threshold still catches actual 90°/180° flips as long as it stays well under those values.
- **RANSAC face-plane corner refinement**, run after the rough ICP pose: fit a plane to each visible *face* (using the low-noise, non-edge points, grouped via pre-existing region-growing clusters rather than raw unsorted points, so the plane-to-face assignment is known), constrain the (up to three) fitted planes to be mutually perpendicular (a cardboard case is a rectangular box), and compute the corner as the intersection of those planes: two planes intersect in an edge line, three planes intersect in a single corner point.
  - RANSAC is needed at this stage because, in clutter (neighboring cases, racking structure, pallet edges surviving a crop), which points belong to which face isn't known with certainty ahead of time even within a region-grown cluster.
  - A depth sensor usually only directly observes 2 of the 3 faces needed for a 3-plane corner — the third plane is borrowed from the known-dimension synthetic template and is only accepted into the solution if it comes out genuinely perpendicular to the two measured planes (i.e. it's a validation-gated synthetic fallback, not injected unconditionally).
- **Template/scene resolution coupling:** `template_resolution_meters` sets both the point spacing used to synthesize the ICP template *and* (via voxel-grid downsampling) the density of the scene's own planar/edge point clouds fed into both ICP and RANSAC. Matching template and scene density is what keeps `icp_fitness_threshold` (a mean-squared-correspondence-distance metric) comparable across different products — if the scene cloud were much denser than the template, fitness would degrade for reasons unrelated to actual alignment quality, making the threshold meaningless across products with different native point densities. `ransac_inlier_percentage` is likewise a fraction measured against this same resolution-matched, downsampled planar cloud.

**Code:**
Rotation gate hard-reject:
```cpp
if (rotation_degrees > config_.expected_rotation_threshold_degrees) {
    RCLCPP_WARN(logger_, "ICP rotation %.1f degrees was greater than expected threshold %f",
        rotation_degrees, config_.expected_rotation_threshold_degrees);
    detection.success = false;
    return detection;
}
```
Matching template/scene density via voxel downsampling:
```cpp
voxel_grid.filter(*edge_point_cloud);
pcl::VoxelGrid<pcl::PointNormal> voxel_grid2;
voxel_grid2.setInputCloud(planar_point_cloud);
voxel_grid2.setLeafSize(
    config_.template_resolution_meters,
    config_.template_resolution_meters,
    config_.template_resolution_meters);
voxel_grid2.filter(*planar_point_cloud);
```

**Gotchas:**
- Precisely stating what the rotation gate measures matters: it's a deviation-from-prior threshold, not a flip-specific detector — conflating the two risks describing the system's actual guarantees incorrectly (e.g. in documentation or an IP disclosure), since the gate would also reject a large non-flip misalignment, and would *not* catch a flip smaller than the configured threshold if one somehow existed for a given product's geometry.
- Fitness score alone cannot distinguish a correct pose from a flipped one on a symmetric-enough face — this is exactly why a separate, independent rotation-vs-prior check exists rather than relying on ICP's own convergence metric.
- The third (synthetic/template) plane in the corner-refinement stage is explicitly validation-gated (must come out perpendicular to the two measured planes) rather than always injected — treating it as unconditionally available would be an inaccurate description of the method's actual robustness.
- `template_resolution_meters` is a single parameter doing double duty (template synthesis density *and* scene downsampling density) — changing it for one purpose (e.g. to speed up processing) silently changes the other, and can shift what `icp_fitness_threshold`/`ransac_inlier_percentage` effectively mean for a given product without those threshold values themselves changing.

**Related:** [[product-mapping-rgbd-segmentation]]

**Still unclear:**
This was reconstructed during a code-reading session specifically to support IP disclosure writing — accuracy was cross-checked against the repo's own `README.md` wording for the rotation-gate semantics, but exact numeric retry logic (e.g. a `runICP_with_sweep` vertical-shift-and-retry mechanism, and how final success is determined after a retry) was being traced in the source session but not resolved before the material was captured — treat that specific retry mechanism as unconfirmed.
