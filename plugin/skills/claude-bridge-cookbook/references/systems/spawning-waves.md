# Spawning / Waves

Release enemies/pickups on a timer up to a live cap, escalating over a round. Built on the real `create_spawner` / `create_npc_spawner` scaffolds, optionally driven by `create_round_phase_machine`.

## What it IS / when you need it

A spawner is three small pieces that ship together: **a cadence gate** (when does the next thing spawn?), **a selection roll** (what spawns?), and **a population cap** (how many at once?). Use it for survival/horde loops, tower-defense waves, or "customers arrive on a curve."

## Bridge build order

**Phase A — generate + compile:**
1. `create_spawner name="EnemySpawner"` — generates a generic prefab Spawner: spawns a prefab on a timer up to a max live count, in a box/sphere area (gizmo-drawn). This is the workhorse.
   - For NPC-specific batches: `create_npc_spawner name="HordeSpawner" count=8 radius=12` (instantiates a prefab at randomized points, all-at-once or on an interval, capped by `maxAlive`).
2. (Optional) `create_weighted_loot_table name="EnemyTable"` if "what spawns" should be a weighted roll (it ships `Roll()` / `TotalWeight` / `RollIndex()`).
3. (Optional, for rounds) `create_round_phase_machine name="WaveDirector" phasesCsv="Prep,Wave,Rest"` — a phase enum with per-phase durations, auto countdown, `StartPhase`/`SkipPhase`, `OnPhaseChanged`. Spawn the wave in the phase-entry hook.
4. `editor_recompile` → `wait_ready` → `get_compile_errors`.

**Phase B — place + wire:**
5. `component_add target="GameManager" type="EnemySpawner"`.
6. Set the prefab to spawn + the cap + the interval via `component_set target="GameManager" componentType="EnemySpawner" propertiesJson="{…}"` (read the real field names first with `get_component_fields`).
7. If using the loot table/round machine, wire them: the spawner asks the table what to spawn; the round machine's `OnPhaseChanged` enables/disables the spawner.

## Verify

- **Edit-scene:** `get_component_fields` confirms the prefab ref, cap, and interval are set. The spawner draws its area gizmo — `frame_object target="GameManager"` + `screenshot_scene` to see where it'll spawn.
- **Runtime:** `sim_play` → poll `sim_status`. Let it run a few seconds, then `find_in_play query="<enemy name>"` — you should see instances appear up to the cap and **stop** (the cap holds). `sim_set_speed speed=4` to fast-forward the cadence so you don't wait. `screenshot_game` to see the spawned crowd; read the PNG. If using the round machine, read its current phase with `get_runtime_property`.

## Variations

- **Escalating cadence:** expose the interval as a field the round machine shortens each phase, or drive it from an `AnimationCurve` over round progress (hand-written in the spawner via `edit_script`).
- **Weighted selection with anti-streak:** `create_weighted_loot_table` gives the weighted pick; for "don't repeat," bump losers' weights and zero the winner after each roll (extend the table or the spawner).
- **Lazy / virtual spawning:** for huge fields, don't instantiate everything — precompute positions and only instantiate when the player is near (hand-written; the scaffold is the eager version).
- **Pickups, not enemies:** the same `create_spawner` with a pickup prefab (`create_pickup`).

## Gotchas

- **Population lists leak** if a spawned object is destroyed without being pruned from the live list — every count read should drop invalid entries. Verify the cap actually holds with repeated `find_in_play` counts.
- **Configure before the object is used.** Set the spawner's prefab + cap fields *before* `sim_play`, and read them back — an unconfigured spawner either does nothing or spawns the wrong thing.
- **Two-phase:** scaffold → recompile → THEN `component_add` + `component_set`.
- **`create_round_phase_machine` re-arms its own timer** — its phases advance on a countdown; don't also hand-tick it. Read `OnPhaseChanged`'s effect via the phase field, not by assuming the call landed.
- **Spawning dozens in one frame stalls** — if you instantiate a big batch, stagger it across frames.

## Cross-links

`references/systems/npc-ai.md` (the enemies that spawn), `references/systems/health-combat.md` (killing them), `references/systems/economy-currency.md` (rewards). Genres: `references/genres/survival.md`, `references/genres/tower-defense.md`, `references/genres/top-down-twin-stick.md`, `references/genres/fps-shooter.md`.
