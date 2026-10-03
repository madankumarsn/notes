<!-- Generative AI was used in the Creation/Modification of this file -->
# Perception Hub

Index of notes in `03-perception/`.

## product-mapping/

- [[product-mapping-rgbd-segmentation]] — pallet-conditioned region growing + plane fitting + dimension validation against a product DB
- [[product-detection-icp-ransac]] — edge-ICP rotation-flip gate + RANSAC face-plane corner refinement
- [[nms-by-prior-box-dims]] — NMS variant ranked by catalog dimension match, suppressed by pixel coverage (not IoU)
- [[product-mapping-on-the-go-alignment]] — blocking alignment wait that captures the exact triggering frame, not the loop-exit frame
- [[product-mapping-tf-publishing]] — when a one-shot detection TF is actually broadcast, RViz frame-timeout trap, missing-intermediate-frame trap
- [[product-mapping-pallet-offset-param]] — applying a fixed pose correction in a stable local frame instead of world frame

## camera/

- [[camera-pose-error-sources]] — extrinsics/timing/localization error terms for a moving-platform camera pipeline, reprojection self-consistency trap, diagnostic protocol
- [[motion-blur-estimation]] — motion blur formula, range-cancellation argument, write-up guidance

## Other

- [[lidar-multi-return-intensity]] — multi-return physical mechanism, intensity-offset encoding, scan geometry
