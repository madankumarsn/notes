---
topic: Where pikaros2's product-mapping node writes its output artifacts
status: draft
layer: projects
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# pikaros2 product-mapping artifact output paths

**One-line idea:**
`product_mapping_online_node.py` can write up to four different kinds of output artifacts (a snapshot rosbag, alignment-band debug images, a cProfile dump, and a viz PNG/TXT pair), all consolidated under one dated root directory per day so a day's worth of mapping runs land together.

**Why it exists:**
A quick-lookup reference for "where did my bag/debug images/profile from today's mapping run actually go" without having to re-read the node's source each time.

**Math / mechanics:**
A single dated root directory (`~/bags/<date>/`) is computed once per day and shared across all four artifact types, via one centralized helper rather than each artifact type formatting its own date independently — so a day's bag, debug images, profile dump, and viz output all land under the same parent regardless of which part of the node produced them. See Code below for the resulting per-artifact paths.

**Code:**
```text
~/bags/<date>/product_mapping/bags/{YYYYMMDD}-{time_sec}_{camera_name}_{requested_product_id}
    # snapshot rosbag (record_bag_snapshot=true); mcap storage; underscores in
    # camera_name are rewritten to hyphens in the folder name
    # topics: RGB, depth, camera_info, /tf, /tf_static, /product_instances

~/bags/<date>/product_mapping/alignment_band/{YYYYMMDD}-{time_sec}_{camera_name}_{requested_product_id}
    # alignment-band debug JPEGs (save_alignment_band_images=true, on-the-go
    # mapping only); files inside named {stamp_ns}_{tag}_ey{error_y:+.4f}_os{0|1}.jpg

~/bags/<date>/product_mapping/product_mapping_profile.prof
    # cProfile dump (profile=true); one file, no per-run timestamp — gets
    # overwritten by the next profiled run

~/bags/<date>/product_mapping/output/...
    # ProcessRGBD's always-on viz dump: {stem}.png + {stem}.txt
    # stem = an explicit artifact_prefix if one was given, otherwise
    # {YYYY-MM-DD_HH-MM-SS}_{time_sec}_{location}_{requested_product}
```

**Gotchas:**
- The **directory** date format is `YYYY-MM-DD` (the dated root under `~/bags/`), but individual **filenames** inside that directory still use the older `{YYYYMMDD}-{time_sec}_...` compact timestamp format — only the directory-level date format was changed; don't expect the same date formatting inside filenames.
- A `bag_name_prefix` parameter exists on the node but is **not actually used** in the bag folder-naming path — a stale code comment describing a `{prefix}__{product_id}` naming scheme was carried over from a different script (`product_mapping_from_long_bag.py`) and doesn't apply to this node's actual behavior. Don't expect setting this parameter to change the bag folder name.
- The helper that builds the dated root directory (`dated_root_dir`) is shared code used by both bag recording and the broader mapping-behavior layer — if the output layout ever looks inconsistent between those two call sites, check whether they're both actually going through this shared helper or whether one has drifted to a separate implementation.

**Related:** [[ros2-bag-corruption-recovery]], [[git-submodules]]

**Still unclear:**
These paths reflect one point-in-time read of the node's source and are specific to this project's current conventions — re-check against the actual code if behavior seems to have changed, since output-path conventions are exactly the kind of thing that gets refactored over time.
