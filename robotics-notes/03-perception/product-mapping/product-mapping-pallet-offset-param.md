---
topic: Applying a configurable pose offset in a stable local frame instead of world frame
status: draft
layer: perception
---
<!-- Generative AI was used in the Creation/Modification of this file -->

# Applying a pose offset in the local (pallet) frame, not world frame

**One-line idea:**
A fixed, configurable pose correction (e.g. shifting a detected product toward the front face of its pallet load) should be applied to the transform expressed in the *stable local frame it's meant to be relative to* (here, the pallet frame) rather than in a world/map frame — that way the correction's meaning and magnitude stay the same regardless of how that pallet happens to be oriented in the world.

**Why it exists:**
A general pattern worth remembering: when a correction is conceptually "move this thing N cm toward feature X of its own container," apply it in the frame where that direction is a fixed axis, not in whatever frame happens to be upstream/downstream in the pipeline — otherwise the same configured offset produces a different real-world shift depending on orientation.

**Math / mechanics:**
Let `T_pallet_box` be a detected product's pose expressed in its pallet's own frame (pallet frame convention: +X points into the load / depth direction). Applying a fixed offset along pallet -X (toward the front face, out of the load) is just a translation added in that frame, independent of `T_world_pallet`:
```
T_pallet_box_shifted.translation = T_pallet_box.translation + [-offset_m, 0, 0]
T_pallet_box_shifted.rotation    = T_pallet_box.rotation        # unchanged
```
Because `T_pallet_box` is computed once (as `T_world_pallet.inv() * T_world_box`) and then consumed by multiple downstream code paths (e.g. both "valid" and "invalid" product-instance construction), applying the offset immediately after that single computation — rather than separately in each downstream path — guarantees every consumer picks it up consistently with no duplicated logic.

**Code:**
```python
# Config: default 0.0 means no behavior change unless explicitly set
product_x_offset_m: float = 0.0
```
```python
T_pallet_box = target_pallet_load.T_world_pallet.inv() * T_world_box

# Shift the product pose toward the pallet's front face by a configurable magnitude
if self.config.product_x_offset_m != 0.0:
    offset = np.array([-self.config.product_x_offset_m, 0.0, 0.0])
    T_pallet_box = T.from_components(
        translation=T_pallet_box.translation + offset,
        rotation=T_pallet_box.rotation,
    )
```

**Gotchas:**
- The config value is specified as a **magnitude**, not a signed direction — a positive `product_x_offset_m` always moves the pose toward the pallet's front face (pallet -X) because the code negates it internally; a reader skimming the sign in the code without reading the comment could easily misinterpret which way a positive value actually shifts the pose.
- Applying the offset at the single point where `T_pallet_box` is first computed (rather than re-applying it separately wherever the pose is later consumed) is what makes it automatically apply to every downstream code path, including ones added later — but it also means any future code path that recomputes `T_pallet_box` independently (instead of reusing this value) would silently skip the offset.

**Related:** [[product-mapping-tf-publishing]]

**Still unclear:**
No rationale for *why* this offset was needed (e.g. compensating a systematic detection bias toward the back of the load, or a physical grasp-clearance requirement) was captured in the source material — only the mechanical implementation. The exact `RigidTransform`/`T` helper class's full API wasn't explored beyond the `from_components` constructor used here.
