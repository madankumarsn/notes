---
topic: Pallet-conditioned RGB-D product mapping via instance segmentation + region growing + plane fitting
status: draft
layer: perception
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# Pallet-conditioned RGB-D product mapping

**One-line idea:**
Localizing pallet-load product cases from a single RGB-D frame combines a learned instance segmentation front-end (box masks) with geometric back-end reasoning (region growing gated by expected face orientation, SVD plane fitting, and dimension validation against a known product/pallet database) rather than trying to infer 3D box geometry from the mask alone.

**Why it exists:**
A concrete example of a broader pattern: using a learned detector only to get a rough 2D region of interest, then handing that region to hand-written geometric reasoning that exploits domain knowledge (known case dimensions, known pallet geometry, gravity direction) the detector itself doesn't have access to — often more robust and more debuggable than pushing the detector to directly output 3D geometry.

**Math / mechanics:**
Pipeline, in order:
1. **Instance segmentation** (Detectron2 Cascade Mask R-CNN) produces box masks; a mask touching the image border is flagged as truncated (partially out of frame).
2. **Back-projection**: valid-depth pixels inside each mask are back-projected to 3D, then voxel-downsampled.
3. **Region growing**, gated by orientation: segmentation uses surface normals (computed via Open3D) plus a radius-neighborhood search (SciPy) and a BFS-style grow (accelerated with Numba), with seeds processed in ascending surface curvature (flattest regions grown first). Growth is restricted to points whose normal aligns with the pallet's known inward-facing axis (the pallet's −X direction, expressed in camera frame) within an angular threshold — this specifically rejects growing onto non-opening faces (e.g. side walls, adjacent cases at an angle) that a naive normal-based region grow would otherwise merge in.
4. **Plane fitting**: each resulting cluster is fit to a plane via SVD (not RANSAC on this path — RANSAC is used in a different, later detection stage, see [[product-detection-icp-ransac]]); clusters whose fit has too low an inlier ratio are discarded outright.
5. **Box construction**: an accepted plane becomes a gravity-aligned oriented box — box +Z is the plane normal pointed toward the camera, +Y is world-up projected onto the plane, and extents come from the planar points' spread. The ratio of the points' convex-hull area to the fitted box's face area is stored as a geometry-quality signal (a low ratio suggests a poor or partial fit).
6. **Dimension validation against a warehouse database**: the known pallet-load volume (pose, case dimensions, rack height from a product/pallet database) gates acceptance, with asymmetric region-of-interest margins per face. Because a single RGB-D view usually only observes two of the three box dimensions directly (height and one horizontal extent), the *other* horizontal dimension (width vs. depth) is inferred by matching the observed horizontal extent against whichever catalog dimension it's closer to, and the unobserved orthogonal size is filled in from the catalog rather than estimated from noisy data.

**Code:**
No production code was quoted verbatim for this pipeline stage (described via architecture walkthrough, not pasted source) — see [[product-detection-icp-ransac]] for the stage with quoted code.

**Gotchas:**
- Detections that fail dimension/tolerance validation are **deliberately still returned** (marked invalid, not dropped) — a downstream depalletizing strategy uses invalid instances as known occluders when planning picks, so discarding them would lose information the rest of the system actively relies on.
- The orientation-gated region growing specifically targets a known failure mode of plain normal-based segmentation: a pallet load's front-facing "opening" surface is what should be grown, and adjacent faces (side walls, neighboring cases) that happen to be locally planar and smoothly connected would otherwise bleed into the same grown region without this gate.
- Side-mounted RGB-D cameras can require a 90° image rotation to match a downstream detector's expected input orientation — if so, the rotation must be composed consistently with intrinsics/extrinsics so the final 3D result still lands correctly in the true (un-rotated) camera frame; getting this composition wrong produces a plausible-looking but spatially wrong result.
- "Mapping on the go" (capturing while the platform is still moving, see [[product-mapping-on-the-go-alignment]]) is a separate concern from this segmentation pipeline itself — the pipeline operates on whatever single RGB-D frame it's given, regardless of how that frame was selected/triggered.

**Related:** [[product-detection-icp-ransac]], [[nms-by-prior-box-dims]], [[product-mapping-tf-publishing]]

**Still unclear:**
This is grounded in a code-reading session done specifically to support writing IP disclosure text, so the description aims for accuracy but wasn't independently re-verified against the current source in this pass — treat file/line specifics as approximate. The source material noted (but didn't detail) a designed-but-not-wired-in idea of using edge-exposure to pick which ICP template to use in a later stage — flagged there as aspirational, not implemented, and not covered further here.
