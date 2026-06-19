# Building / Placement

A ghost-then-commit placement system: preview a building under the cursor, validate, and place it on confirm. The bridge has no single "placement" scaffold, so this is mostly hand-written C# — but the bridge's raycast/overlap/snap tools do the spatial heavy lifting and let you verify the math before you trust it.

## What it IS / when you need it

Placement = **a ghost** (a semi-transparent preview that follows the cursor on the ground), **validation** (is this spot clear / on valid ground / affordable?), and **commit** (instantiate the real building, spend currency). Reach for it for tycoons, base-builders, tower defense (placing towers), farming plots.

## Bridge build order

**Phase A — generate + compile:**
1. The buildings themselves can reuse scaffolds: a producer building might carry a `create_economy_wallet`-fed income tick or an `create_interactable`. Generate those first.
2. Hand-write a `PlacementController` (`create_script`): holds the selected building prefab, casts a ground ray each frame, moves the ghost, checks validity, and on confirm instantiates + charges. The bridge has no `create_placement_mode` tool, so this Component is yours.
3. `editor_recompile` → `wait_ready` → `get_compile_errors`.

**Phase B — place + wire + validate the math with bridge tools:**
4. `component_add target="GameManager" type="PlacementController"`; give it a building prefab (`prefab_find` / a prefab you made) and the ground layer.
5. **Use the bridge to nail the spatial logic before relying on the in-code version:**
   - `raycast origin="<camera>" direction="<down/forward>" maxDistance=1000` returns the ground hit point/normal — confirm your cursor→ground projection lands where you expect.
   - `snap_to_ground target="Ghost"` drops the ghost onto the surface below it (offset/startHeight/maxDistance params) — the same operation your controller does each frame, so you can eyeball it.
   - `physics_overlap_box center="<ghost pos>" halfExtents="<building half-size>"` lists GameObjects whose colliders overlap the footprint — this IS your "is the spot clear?" check; run it at a candidate position and see what it reports.
6. Wire commit: on confirm, `TrySpend` the cost (see `references/systems/economy-currency.md`), instantiate the real prefab at the validated transform, and record it for save (`references/systems/save-load.md`).

## Verify

- **Spatial math, before play:** run `raycast` and `physics_overlap_box` at a few positions and confirm they return what your validation expects (clear vs overlapping). This is the highest-leverage check — placement bugs are almost always bad raycasts or overlap tests, and the bridge lets you test them in isolation.
- **The ghost, visually:** `frame_object target="Ghost"` + `screenshot_scene`, read the PNG — is it sitting flush on the ground (`snap_to_ground`) and tinted as a preview?
- **Runtime:** `sim_play` → poll `sim_status`. Drive a placement: position the ghost (`set_runtime_property`/`teleport`), `invoke_method target="GameManager" componentType="PlacementController" methodName="<Confirm/Place>"`, then `find_in_play query="<building>"` to confirm one was created and `get_runtime_property` the wallet to confirm it charged. Test the reject path: place on an occupied tile and confirm nothing spawns and money is unchanged.

## Variations

- **Grid snap:** round the raycast hit to a grid before placing — store buildings by grid cell to test occupancy with a lookup instead of an overlap query (faster for dense grids).
- **Rotation:** quarter-turn the ghost; swap the footprint's X/Z extents when rotated before the overlap test.
- **Footprint validity tint:** swap the ghost's material color (`material_set_color target="Ghost" color="0,1,0,0.5"` valid / `"1,0,0,0.5"` invalid) based on the overlap result.
- **Demolish:** the inverse — raycast to a placed building, refund a fraction, `game_object_destroy`.

## Gotchas

- **`physics_overlap_box` only sees colliders** — a building footprint with no collider won't be detected as occupying space, and validation will let you stack. Confirm colliders with `get_component_info`.
- **`snap_to_ground` traces straight down** from above the object — if the ghost starts below the surface or the trace is too short, it won't land; tune `startHeight`/`maxDistance`.
- **Charge on commit, never on preview**, and through `TrySpend` so you can't place what you can't afford.
- **Two-phase:** generate the controller + building scaffolds → recompile → THEN `component_add` + wire.
- **Persist relative/grid coordinates**, not just world positions, so a placed layout reloads correctly — re-instantiate on load (`references/systems/save-load.md`).

## Cross-links

`references/systems/economy-currency.md` (paying), `references/systems/save-load.md` (persisting layouts), `references/genres/tycoon-management.md` (producers), `references/genres/tower-defense.md` (placing towers). Spatial tools: the Worldgen/Scatter/Physics family in `docs/TOOLS.md`.
