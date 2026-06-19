# Puzzle — Genre Recipe

Rooms of triggers, switches, and gates that must be solved in order to progress. Built from the trigger-zone, interactable, objective, and game-manager scaffolds — light on physics, heavy on state.

## The loop

Enter a room → observe the puzzle → interact (push buttons, move objects, hit switches, step on pads) → satisfy a condition → a gate opens / the next room unlocks → reach the exit. Correctness is **logical state**, which the bridge verifies cleanly (unlike feel-heavy genres).

## The system stack to compose

1. **Interactables + triggers** — `create_interactable` (switches/buttons) and `create_trigger_zone` (pressure pads, room entries) (`references/systems/objectives-quests.md` covers triggers).
2. **Puzzle state / win** — `create_game_manager name="GameManager" statesCsv="Playing,Solved"` and/or `create_objective_system` to track sub-goals.
3. **Movable objects** — primitives with `physics_add_rigidbody` for push/weight puzzles.
4. **Gates/doors** — objects toggled active/animated when the condition is met.
5. **HUD** — hints, step counter (`references/systems/ui-hud.md`).

## Bridge build order

**1. Block out the room(s).** Walls, a gate, switches, pads — all primitives. `screenshot_scene`, read the layout.

**2. Phase A — generate, then compile:**
- `create_player_controller name="Player" mode="first_person"` (or third/top-down to taste)
- `create_interactable name="Switch" prompt="Press E" range=2.5`
- `create_trigger_zone name="PressurePad"`
- `create_objective_system name="Steps"` (track "switches thrown") and/or `create_game_manager name="GameManager" statesCsv="Playing,Solved"`
- a small `PuzzleController` script (`create_script`) that watches the switches/pads and opens the gate when the condition holds
- `editor_recompile` → `wait_ready` → `get_compile_errors`.

**3. Phase B — place + wire:**
- Player tagged `Player` (`add_tag` → `game_object_set_tag`) so pad triggers filter correctly.
- Each `Switch` toggles a bool in `PuzzleController`; each `PressurePad` `OnEnter`/`OnExit` (filtered to `Player` or a movable box's tag) sets/clears its bool.
- `PuzzleController` checks the combination; when satisfied, open the gate (`game_object_set_active` off, or animate) and `GameManager.ChangeState(Solved)` / complete the objective.
- Movable boxes: `physics_add_rigidbody` so the player can push them onto pads.

## How to verify each step

- **State logic is fully bridge-verifiable** (the genre's gift): `sim_play`. Throw a switch directly: `invoke_method target="Switch" componentType="Interactable" methodName="Interact"` (or set the controller's bool via `set_runtime_property`), then `get_runtime_property target="GameManager" componentType="PuzzleController" propertyPath="<solved flag>"`. Satisfy the full combination and confirm the gate opened (`get_runtime_property` the gate's active state, or `find_in_play`) and the state flipped to `Solved`.
- **Pads:** `teleport target="Player" position="<on the pad>"` and confirm the pad's bool set; `teleport` off and confirm it cleared.
- **Layout/gate visually:** `frame_object target="Gate"` + `screenshot_scene` before/after solving; read the PNGs.
- **Movable boxes:** push physics is felt by the human, but you can confirm a box on a pad sets the bool via the runtime read.

## Gotchas

- **Pads need the player (or box) tagged + a trigger collider** — a pad that never registers is a missing tag or a solid collider (`get_component_info`).
- **`create_trigger_zone` once-only by default** — a re-usable pad must re-fire on exit/re-enter; don't set the once flag for those.
- **Check the win condition on state change, not in `Update`** where possible — and verify by driving the inputs via `invoke_method`/`set_runtime_property`, not by assuming.
- **Interact range too small** = unsolvable; confirm with `get_component_fields`.
- **Two-phase** for the controller + scaffolds.

## Cross-links

Systems: objectives-quests (triggers/goals), ui-hud (hints), building-placement (for sokoban-style grids). `create_game_manager` for room/level state. Workflow: `claude-bridge-build-feature`.
