---
topic: Non-maximum suppression keyed on catalog dimension match and pixel coverage, not score/IoU
status: draft
layer: perception
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# NMS by prior-box dimension match and pixel coverage

**One-line idea:**
A non-standard NMS variant that sorts candidate masks by how well they match a *known catalog dimension* (not detector confidence score) and suppresses based on what fraction of a candidate's own pixels are already claimed by a higher-ranked kept mask (pixel **coverage**, not pairwise IoU) — catching both a small mask nested entirely inside a larger kept one and a large mask that wraps several kept ones, cases where ordinary IoU-based NMS would leave a bad box untouched.

**Why it exists:**
A reusable NMS variant pattern for any detector where you have external ground-truth-like priors (e.g. known object dimensions from a catalog/database) to rank candidates by, and where overlapping detections can be nested or wrap-around rather than simple side-by-side overlaps that pairwise IoU handles well.

**Math / mechanics:**
- **Ranking metric (lower is better):** `dim_error = |height_err| + min(|width_err|, |depth_err|)`, where each `*_err` is the candidate's measured dimension minus the expected catalog dimension along that axis. Candidates outside the region of interest never get a real dimension measurement and are assigned `dim_error = inf`, which sorts them strictly last (never preferred, but not discarded outright either).
- **Suppression rule**, applied to candidates in best-first (lowest `dim_error`) order: for each candidate, compute
  \[
  \text{coverage} = \frac{\text{pixels already claimed by a previously-kept mask}}{\text{candidate's own mask area}}
  \]
  Drop the candidate if `coverage > mask_coverage_thresh` (a tunable, default `0.7`) or if its mask is empty; otherwise **keep** it and mark its still-unclaimed pixels as now claimed, for subsequent candidates to be checked against.
- **Why coverage instead of pairwise IoU against each already-kept mask:** IoU between a small mask and a large kept mask that fully contains it stays low (the union is dominated by the large mask), so IoU-based NMS would never suppress the nested small mask. Likewise a single large mask that spans across several already-kept smaller masks has low pairwise IoU against any *one* of them individually. Coverage instead asks "how much of *this candidate* is already explained by what's kept so far," which catches both nested-inside and wraps-around-multiple cases directly.
- **A later/lower-ranked candidate can still survive** if most of its own pixels were never claimed by any earlier-kept mask — this is intentional: a region that no good-dimension-match box explains stays visible as its own candidate, rather than being deleted outright just because it overlaps a higher-ranked (but dimensionally worse) winner somewhere.

**Code:**
No production code was quoted verbatim in the source material beyond the metric/rule described above (this was explained via flowchart + narrative, not a pasted implementation).

**Gotchas:**
- This runs as an explicit **second pass** after a primary detection/segmentation stage, gated by a single threshold parameter (`mask_coverage_thresh` in the example this was drawn from) — setting that threshold `<= 0` disables the whole pass, a useful quick on/off switch when comparing "with vs. without this NMS variant."
- It only considers instances whose `id` maps to a valid mask index — including invalid/out-of-ROI candidates, which are kept in the ranking (sorted last via `inf`) rather than removed before this stage runs, since a later stage may still want to see/use them (e.g. as occluders).
- Because ranking is by *catalog dimension match* rather than detector confidence, a visually strong (high-confidence) but dimensionally-wrong detection can be suppressed in favor of a weaker-confidence detection that happens to match the known product's real-world size — this NMS variant is deliberately not "trust the detector's own score."

**Related:** [[product-mapping-rgbd-segmentation]]

**Still unclear:**
General algorithmic structure is grounded in a description of one project's actual implementation; the exact production code (file/line) wasn't quoted verbatim, only reconstructed from a flowchart-plus-explanation session. Behavior on edge cases not discussed (e.g. ties in `dim_error`, multiple candidates with identical coverage) wasn't captured in the source material.
