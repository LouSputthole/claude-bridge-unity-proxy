# FPS / Shooter — Genre Recipe

A first-person controller, weapons that deal damage (hitscan via `raycast` or projectiles), enemies that path and fight, and a wave/spawn flow. Built from the player-controller, health, npc-brain, and spawner scaffolds.

## The loop

Move + aim (first-person) → shoot enemies → enemies path to you and attack → kill them for score/loot → survive escalating waves → win/lose. The whole genre hangs off "a shot reaches a target's Health."

## The system stack to compose

1. **First-person controller + camera** — `create_player_controller mode="first_person"`.
2. **Player + enemy health** — `create_health_system` (`references/systems/health-combat.md`).
3. **Weapons** — hitscan (`Physics.Raycast`, prototyped with the bridge `raycast` tool) or projectile prefabs (`physics_add_rigidbody`). See `references/systems/health-combat.md`.
4. **Enemy AI** — `create_npc_brain preset="predator"` + `bake_navmesh` (`references/systems/npc-ai.md`).
5. **Spawning / waves** — `create_spawner` / `create_npc_spawner`, optionally `create_round_phase_machine` (`references/systems/spawning-waves.md`).
6. **HUD** — crosshair, health bar, ammo (`references/systems/ui-hud.md`).
7. **Loot** (optional) — `create_weighted_loot_table` on enemy death.

## Bridge build order

**1. Block out an arena.** Ground + cover boxes (`game_object_create_primitive`). `screenshot_scene`, read framing.

**2. Phase A — generate, then compile:**
- `create_player_controller name="FpsPlayer" mode="first_person"`
- `create_health_system name="Health" maxHealth=100`
- `create_npc_brain name="Grunt" preset="predator"`
- `create_spawner name="EnemySpawner"` (+ `create_round_phase_machine name="WaveDirector" phasesCsv="Prep,Wave,Rest"` if you want rounds)
- a weapon script via `create_script` (hitscan: cast a ray on fire, `TakeDamage` the hit's Health)
- `editor_recompile` → `wait_ready` → `get_compile_errors`.

**3. Phase B — bake nav, place, wire:**
- Player body: `component_add type="FpsPlayer"` + `type="Health"`, attach the weapon script, `add_audio_listener` to the camera.
- **`bake_navmesh`** (confirm `get_navmesh_path`); enemy prefab gets `add_navmesh_agent` + `Grunt` brain + `Health`.
- Wire `EnemySpawner` to the enemy prefab; the round machine's `OnPhaseChanged` enables it during Wave.
- Enemy `Health.OnDeath` → score + optional loot roll.
- HUD: `create_canvas` + crosshair `create_ui_image` + health `create_ui_slider` bound to `HealthFraction`.

## How to verify each step

- **Aim/shot geometry:** `raycast origin="<muzzle>" direction="<camera forward>" maxDistance=100` — confirm it reaches a target before trusting the in-code cast. `hit:false` = missing collider or wrong direction, not a damage bug.
- **Damage:** can't reliably mouse-aim in play mode, so prove the weapon→health path with `invoke_method target="<enemy>" componentType="Health" methodName="TakeDamage" argsJson="[34]"` → read `HealthFraction`; drive to death and confirm `OnDeath` (despawn/score).
- **Enemy pathing:** `bake_navmesh` + `get_navmesh_path`; `sim_play`, `teleport` the player into view, read the brain flipping to Chase; `screenshot_game`.
- **Waves:** `sim_play`, `sim_set_speed speed=4`, `find_in_play query="<enemy>"` rises to the cap then holds; read the round phase via `get_runtime_property`.
- **HUD:** `screenshot_game`, read crosshair + health bar; `TakeDamage` the player and confirm the bar moves.

## Gotchas

- **No NavMesh = enemies stand still** — bake first, re-bake after geometry changes.
- **Hitscan needs colliders on targets** — verify with `get_component_info`; the bridge `raycast` returning false is your tell.
- **Move projectiles via Rigidbody, fire in `Update`, collide in `FixedUpdate`** — raw transform sets skip collisions.
- **The bridge can't judge gunfeel** (recoil, fire rate, hit-stop) — wire logic, verify numbers, hand the human a playtest.
- **Two-phase** for every scaffold; configure the spawner before `sim_play`.

## Cross-links

Systems: health-combat, npc-ai, spawning-waves, ui-hud, progression-upgrades (weapon upgrades). Cousins: `references/genres/top-down-twin-stick.md` (same combat, top-down), `references/genres/survival.md`. Workflow: `claude-bridge-build-feature`.
