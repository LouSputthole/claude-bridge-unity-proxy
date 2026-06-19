# Top-Down / Twin-Stick Shooter — Genre Recipe

A top-down mover that aims and shoots in any direction while enemies swarm. Same combat spine as an FPS, top-down camera. Built from the player-controller (`top_down`), health, spawner, and loot scaffolds.

## The loop

Move (top-down) + aim/shoot in any direction → enemies swarm toward you → kill them for XP/loot → survive escalating waves → upgrade → repeat. A bullet-heaven/roguelite is this loop with auto-fire + draft upgrades.

## The system stack to compose

1. **Top-down controller** — `create_player_controller mode="top_down"`.
2. **Player + enemy health** — `create_health_system` (`references/systems/health-combat.md`).
3. **Weapon** — projectile or hitscan toward the aim point (`create_script`; prototype geometry with `raycast`).
4. **Swarm spawning** — `create_spawner` / `create_npc_spawner` (`references/systems/spawning-waves.md`), enemies use `create_npc_brain preset="predator"` (NavMesh optional for simple "move toward player").
5. **Loot / XP** — `create_weighted_loot_table` on death, `create_pickup` for drops (`references/systems/inventory.md`).
6. **Progression** — `create_stat_modifier_system` for the draft/upgrade between waves (`references/systems/progression-upgrades.md`).
7. **HUD** — health, XP, wave counter (`references/systems/ui-hud.md`).

## Bridge build order

**1. Block out an arena.** Ground plane + walls (`game_object_create_primitive`); place a top-down camera looking straight down (`transform_set_rotation target="Main Camera" euler="90,0,0"`, lift it on Y, then `screenshot_game` to confirm the top-down view).

**2. Phase A — generate, then compile:**
- `create_player_controller name="Player" mode="top_down"`
- `create_health_system name="Health" maxHealth=100`
- `create_npc_brain name="Swarmer" preset="predator"`, `create_spawner name="Swarm"`
- `create_weighted_loot_table name="DropTable"`, `create_pickup name="XpOrb" kind="xp"`
- `create_stat_modifier_system name="RunStats" statsCsv="Damage,FireRate,MoveSpeed"`
- a weapon script via `create_script`
- `editor_recompile` → `wait_ready` → `get_compile_errors`.

**3. Phase B — place + wire:**
- Player: body + `component_add type="Player"` + `type="Health"` + `type="RunStats"` + the weapon; `add_audio_listener` to the camera.
- Enemy prefab: `Swarmer` brain + `Health`; if using NavMesh, `bake_navmesh` + `add_navmesh_agent` (or a simple "translate toward player" if you skip nav).
- Wire `Swarm` to the enemy prefab; enemy `Health.OnDeath` → `DropTable.Roll()` → spawn an `XpOrb`.
- Between waves: a draft picks a `RunStats` modifier (more Damage/FireRate).
- HUD bound to Health + XP + wave count.

## How to verify each step

- **Top-down camera:** `screenshot_game` — read the PNG; the view should look straight down. Adjust the camera euler/height until it does.
- **Combat path:** prove weapon→damage with `invoke_method target="<enemy>" componentType="Health" methodName="TakeDamage" argsJson="[25]"` → read `HealthFraction`; drive to death → confirm a loot orb appears (`find_in_play query="XpOrb"`).
- **Swarm:** `sim_play`, `sim_set_speed speed=4`, `find_in_play query="<enemy>"` rises to cap; `screenshot_game` to see the swarm; read the PNG.
- **Upgrade:** `invoke_method` the draft pick → `get_runtime_property` the stat rose → confirm faster kills.
- **Movement/aim feel:** `screenshot_game` confirms it renders; hand the human the playtest for responsiveness.

## Gotchas

- **Top-down camera setup is fiddly** — verify with a `screenshot_game` and read it; don't assume the euler/height are right.
- **Many enemies = perf** — cap the spawner, prune dead from the live list, stagger big spawns across frames.
- **Simple chase vs NavMesh:** for open arenas a "translate toward player" enemy is fine and skips baking; use NavMesh only if there are real obstacles to path around.
- **Move projectiles via Rigidbody**; aim toward the cursor/stick direction in code.
- **Feel (twin-stick aim, dodge) is a human call** — wire it, then playtest with the user.

## Cross-links

Systems: health-combat, spawning-waves, npc-ai, progression-upgrades, inventory (loot), ui-hud. Same combat as `references/genres/fps-shooter.md` (different camera). Workflow: `claude-bridge-build-feature`.
