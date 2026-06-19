# Survival — Genre Recipe

Manage depleting vitals while you gather, craft, and fend off threats — usually under a day/night clock. Built from the health, inventory, pickup, spawning, and save scaffolds.

## The loop

Spawn into a hostile space → manage depleting vitals (health + stamina/hunger) → gather resources (pickups / nodes) → craft/upgrade → survive escalating threats (often at night) → persist progress. Horror is *pacing of pressure*; survival is *replenishment vs. drain*.

## The system stack to compose

1. **Player controller + camera** — `create_player_controller mode="first_person"` (or `third_person`).
2. **Vitals** — `create_health_system` for HP; for stamina/hunger, either extra `create_health_system` instances (each is a clamped pool with events) or a `create_stat_modifier_system` with depleting values. See `references/systems/health-combat.md` + `references/systems/progression-upgrades.md`.
3. **Inventory + pickups** — `create_inventory` + `create_pickup` (`references/systems/inventory.md`).
4. **Threats** — `create_npc_brain preset="predator"` + `create_spawner` (`references/systems/npc-ai.md`, `references/systems/spawning-waves.md`).
5. **Day/night** — `create_day_night_clock` so threats escalate at night (`references/systems/day-night.md`).
6. **Save** — `create_save_system` (`references/systems/save-load.md`).
7. **HUD** — vitals bars + inventory (`references/systems/ui-hud.md`).

## Bridge build order

**1. Block out + atmosphere.** Ground + obstacles, then `apply_atmosphere mood="overcast"` (or `foggy-dawn`) for mood. `screenshot_scene`, read it.

**2. Phase A — generate, then compile:**
- `create_player_controller name="Survivor" mode="first_person"`
- `create_health_system name="Health" maxHealth=100`, `create_health_system name="Stamina" maxHealth=100` (reuse the pool for any depleting vital)
- `create_inventory name="Backpack" slots=20`, `create_pickup name="ResourcePickup" kind="wood"`
- `create_npc_brain name="Predator" preset="predator"`, `create_spawner name="ThreatSpawner"`
- `create_day_night_clock name="DayClock" dayLengthSeconds=180`, `create_save_system name="SaveSystem"`
- `editor_recompile` → `wait_ready` → `get_compile_errors`.

**3. Phase B — bake nav, place, wire:**
- Build the player: a body, `component_add type="Survivor"` + `type="Health"` + `type="Stamina"` + `type="Backpack"`; add an `add_audio_listener` to its camera.
- **`bake_navmesh`** so predators can path (confirm with `get_navmesh_path`).
- Threats: a predator prefab with `add_navmesh_agent` + `Predator` brain; wire the `ThreatSpawner` to it, gated to spawn more at night (`DayClock.OnDusk`).
- Vitals drain: a small script ticking Stamina down on sprint and Health down when hunger bottoms out (`edit_script`/`create_script`); pickups `Backpack.Add`.
- HUD + save as in the system refs.

## How to verify each step

- **Controller:** `sim_play` → `screenshot_game`; hand the human a movement playtest (the bridge can't feel it). It CAN confirm the controller exists and the camera renders.
- **Vitals:** `invoke_method target="Survivor" componentType="Health" methodName="TakeDamage" argsJson="[40]"` → `get_runtime_property … HealthFraction` ≈ 0.6. Same for Stamina.
- **Pickup:** `teleport` the player onto a `ResourcePickup`; confirm `Backpack` count rose and the pickup is gone (`find_in_play`).
- **Threats:** `bake_navmesh` then `get_navmesh_path` proves pathing; `sim_play`, teleport the player near a predator, read the brain's state flipping to Chase; `screenshot_game`.
- **Night escalation:** `sim_set_speed speed=20`, screenshot the lighting swing, confirm more threats spawn after dusk (`find_in_play` count rises).
- **Save:** round-trip vitals + inventory (see `references/systems/save-load.md`).

## Gotchas

- **No NavMesh = predators don't move** — bake before expecting them to hunt.
- **Vitals must clamp** `[0,max]` and drain via `Time.deltaTime` or a fixed tick — verify the fraction read-back, not the call.
- **Inches vs. units** only matters if you mix systems; keep distances consistent.
- **Atmosphere tools are Built-in RP** — `apply_atmosphere`/`set_fog` apply correctly here.
- **Difficulty/feel is a human call** — wire the drain rates, then have the user playtest the pressure curve.

## Cross-links

Systems: health-combat, inventory, spawning-waves, npc-ai, day-night, save-load, ui-hud. Close cousin: `references/genres/horror.md` (lean harder on atmosphere + a single stalker). Workflow: `claude-bridge-build-feature`.
