# Platformer / Obstacle — Genre Recipe

A jumping mover through a course of hazards and checkpoints to a goal. Built from the player-controller scaffold + trigger zones + objectives, with a camera that follows.

## The loop

Move + jump through a level → avoid hazards / time platforms → hit checkpoints → reach the goal trigger to win (or fall/die and respawn at the last checkpoint). Correctness lives heavily in **feel** — jump arc, coyote time, camera lag — which is a human call.

## The system stack to compose

1. **Controller with jump** — `create_player_controller mode="third_person"` (third-person is the usual platformer camera; `top_down` for a top-down platformer).
2. **Hazards + checkpoints + goal** — `create_trigger_zone` (`references/systems/objectives-quests.md` covers triggers).
3. **Win/lose state** — `create_objective_system` (reach-the-goal) and/or `create_game_manager` for Playing/Won/Lost.
4. **HUD** — timer, deaths, checkpoint feedback (`references/systems/ui-hud.md`).
5. **Camera follow** — a follow script in `LateUpdate` (hand-written; the bridge can frame it but follow-feel is yours to tune).

## Bridge build order

**1. Block out the course.** A start platform, gaps, moving-platform stand-ins, and a goal — all `game_object_create_primitive`. Use `grid_duplicate` / `distribute_objects` to lay out repeating platforms fast, `align_objects` to line them up. `screenshot_scene`, read the layout.

**2. Phase A — generate, then compile:**
- `create_player_controller name="Hero" mode="third_person"`
- `create_trigger_zone name="Hazard"` (kills/respawns), `create_trigger_zone name="Checkpoint"`, `create_trigger_zone name="Goal"`
- `create_objective_system name="Objectives"` (and/or `create_game_manager name="GameManager" statesCsv="Playing,Won,Lost"`)
- `editor_recompile` → `wait_ready` → `get_compile_errors`.

**3. Phase B — place + wire:**
- Player: a body + `component_add type="Hero"`; tag it `Player` (`add_tag` → `game_object_set_tag`) so the trigger filters match.
- Place hazards/checkpoints/goal zones; each `OnEnter` (filtered to `Player`) does its job: Hazard → respawn at last Checkpoint; Checkpoint → store respawn point; Goal → complete the objective / `GameManager.ChangeState(Won)`.
- Camera follow script parented logic in `LateUpdate`.
- HUD: a timer + death counter (`references/systems/ui-hud.md`).

## How to verify each step

- **Layout:** `screenshot_scene` / `frame_object` per platform group; read the PNGs — gaps look jumpable? Measure with `measure_distance a="PlatformA" b="PlatformB"` to sanity-check jump distances.
- **Triggers without playing:** confirm each zone has a trigger collider and the right tag filter (`get_component_info`). Then `sim_play`, `teleport target="Hero" position="<inside Checkpoint>"`, and confirm the respawn point updated (`get_runtime_property`); `teleport` into Hazard and confirm a respawn; `teleport` into Goal and confirm the win state (`get_runtime_property` the GameManager/objective).
- **Movement/jump feel:** `sim_play`, `screenshot_game` to confirm the player renders and the camera frames it — then **hand the human the playtest**: "is the jump height/arc right, does the camera lag too much?" The bridge can't feel it. You can best-effort `simulate_input action="Jump"` / `press_and_hold action="Jump" seconds=0.3` to nudge it, but feel is theirs.

## Gotchas

- **Triggers need the player tagged + a trigger collider on the zone** — a goal that never fires is almost always a missing tag or a solid (non-trigger) collider.
- **`create_trigger_zone` is once-only by default** — fine for a goal, but a reusable checkpoint/hazard needs re-fire (reset the once flag or don't set it).
- **Jump physics in `FixedUpdate`, input in `Update`** — reading jump in `FixedUpdate` drops inputs.
- **Camera follow belongs in `LateUpdate`** so it tracks after the player moves; otherwise it jitters.
- **The bridge cannot tune feel** — coyote time, jump buffering, camera damping are human-playtested; wire sensible defaults and iterate with the user.

## Cross-links

Systems: objectives-quests (triggers, goal), ui-hud (timer), progression-upgrades (collectibles → unlocks). For top-down movement instead, `references/genres/top-down-twin-stick.md`. Workflow + the "hand feel to the human" section: `claude-bridge-build-feature`.
