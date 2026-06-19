# Racing / Driving — Genre Recipe

A vehicle around a track through checkpoints, timed laps, finish line. Built from a vehicle controller (Rigidbody-driven), trigger-zone checkpoints, and the round-phase machine — physics-heavy, so verify the layout and lean on the human for handling feel.

## The loop

Drive a vehicle → pass ordered checkpoints (so you can't shortcut) → complete laps → beat a time / opponents → finish. Vehicle *handling* (grip, accel, drift) is feel-dependent and a human call; the track logic (checkpoints, laps, timing) is bridge-verifiable.

## The system stack to compose

1. **Vehicle controller** — start from `create_player_controller` (then adapt to a Rigidbody car) or hand-write a Rigidbody vehicle; the body needs `physics_add_rigidbody` + colliders. (`describe_type WheelCollider` if you want Unity's wheel physics — verify it's available in your version first.)
2. **Checkpoints + finish** — `create_trigger_zone` (ordered) (`references/systems/objectives-quests.md` covers triggers).
3. **Lap/race flow + timing** — `create_round_phase_machine name="RaceDirector" phasesCsv="Countdown,Racing,Finished"` + a lap/timer script.
4. **HUD** — lap counter, timer, speed (`references/systems/ui-hud.md`).
5. **Opponents** (optional) — `create_npc_brain` + `place_patrol_route` along the racing line, or NavMesh.

## Bridge build order

**1. Block out the track.** A ground plane + a looping series of walls/barriers; place checkpoint gates around the loop with `place_along_path` or by hand, a finish gate at the start/finish. `screenshot_scene` (or a top-down `screenshot_game`), read the layout.

**2. Phase A — generate, then compile:**
- the vehicle script (`create_player_controller name="Car"` as a base, or a `create_script` Rigidbody car)
- `create_trigger_zone name="Checkpoint"` (you'll place several, ordered) + `create_trigger_zone name="Finish"`
- `create_round_phase_machine name="RaceDirector" phasesCsv="Countdown,Racing,Finished"`
- a `LapTracker` script (counts ordered checkpoints, increments lap on finish-after-all-checkpoints, tracks time)
- `editor_recompile` → `wait_ready` → `get_compile_errors`.

**3. Phase B — place + wire:**
- Car: body + `physics_add_rigidbody mass=1200` + colliders + the vehicle script; tag it `Player`; `add_audio_listener` on its camera.
- Place checkpoints in order around the loop (give each an index); the `Finish` gate too. Each `OnEnter` (filtered to `Player`) reports its index to `LapTracker`.
- `LapTracker` only counts a lap if all checkpoints were hit in order (anti-shortcut), then bumps the lap and resets the checkpoint set.
- `RaceDirector` runs Countdown → Racing → Finished; finishing the last lap → Finished.
- HUD bound to lap count + timer.

## How to verify each step

- **Track layout:** a top-down `screenshot_game` (lift + point the camera down) or `screenshot_scene`; read the PNG — is the loop closed, are checkpoints spaced around it? `measure_distance` between gates to sanity-check spacing.
- **Checkpoint/lap logic (bridge-verifiable):** `sim_play`. `teleport target="Car" position="<at checkpoint 0>"` then `<checkpoint 1>` … in order, and read `get_runtime_property target="..." componentType="LapTracker" propertyPath="<lap/checkpoint index>"` — the lap should only increment after all checkpoints + the finish, in order. Test the anti-shortcut: `teleport` straight to `Finish` skipping checkpoints and confirm the lap does NOT count.
- **Race flow:** read `RaceDirector`'s phase via `get_runtime_property`; confirm Countdown→Racing→Finished.
- **Handling/feel is a human call** — `sim_play` + `screenshot_game` confirms the car renders and moves; whether it grips/drifts/accelerates right is theirs to playtest. You can best-effort `press_and_hold action="Accelerate" seconds=2` to nudge it.

## Gotchas

- **Checkpoint ORDER is the anti-shortcut** — a lap that counts when you skip gates means the order check is wrong; verify with the teleport-skip test above.
- **Vehicle physics needs a Rigidbody + good colliders** — a car with a bad collider tunnels through walls; confirm `get_component_info`.
- **Move the car via the Rigidbody (forces/velocity) in `FixedUpdate`** — raw transform-setting fights the physics and breaks collisions.
- **`WheelCollider` may not suit every project** — verify it exists (`describe_type`) and is worth the complexity; a simpler force-based car is often enough.
- **The bridge can't tune handling feel** — wire sensible defaults, then iterate grip/accel/drift with the user's playtest.

## Cross-links

Systems: objectives-quests (checkpoint triggers), ui-hud (lap/timer), spawning-waves/npc-ai (AI opponents). `create_round_phase_machine` for the race flow. Workflow + the feel-is-human section: `claude-bridge-build-feature`.
