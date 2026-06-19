# Tower Defense — Genre Recipe

Enemies path along a route to your base; you place towers that shoot them; you earn currency to build more, in escalating waves. Built from the spawner, NavMesh, economy, health, and round-machine scaffolds + the placement pattern.

## The loop

Waves of enemies spawn and **path to your base/goal** → you spend currency to **place towers** along the route → towers auto-target and damage enemies → kills pay currency → survive N waves (lose life when an enemy reaches the goal). 

## The system stack to compose

1. **Enemy pathing** — `bake_navmesh` + `add_navmesh_agent`, or a fixed waypoint path (`place_patrol_route`). Enemies path to the goal (`references/systems/npc-ai.md`).
2. **Wave spawning** — `create_spawner` / `create_npc_spawner` driven by `create_round_phase_machine` (`references/systems/spawning-waves.md`).
3. **Currency** — `create_economy_wallet` (`references/systems/economy-currency.md`).
4. **Tower placement** — the ghost-then-commit pattern (`references/systems/building-placement.md`), towers cost currency.
5. **Enemy + base health** — `create_health_system` (`references/systems/health-combat.md`).
6. **HUD** — currency, lives, wave counter (`references/systems/ui-hud.md`).

## Bridge build order

**1. Block out the map + path.** Ground + a winding route of obstacles; a goal marker at the end. Either `bake_navmesh` (enemies path goal-ward) or `place_patrol_route name="Path" pointsJson="[…]"` for a fixed lane. `screenshot_scene`, read it.

**2. Phase A — generate, then compile:**
- `create_health_system name="EnemyHealth" maxHealth=50`, `create_health_system name="BaseHealth" maxHealth=20` (lives)
- `create_economy_wallet name="Wallet" currencyName="Credits" startingBalance=100`
- `create_npc_brain name="Crawler" preset="wanderer"` (it just needs to path to the goal; the brain handles movement)
- `create_spawner name="WaveSpawner"`, `create_round_phase_machine name="WaveDirector" phasesCsv="Build,Wave,Reward"`
- a `Tower` script via `create_script` (acquire nearest enemy in range, `TakeDamage` on a fire interval) + a `PlacementController` (see `references/systems/building-placement.md`)
- `editor_recompile` → `wait_ready` → `get_compile_errors`.

**3. Phase B — bake nav, place, wire:**
- `bake_navmesh` (confirm `get_navmesh_path` from spawn to goal is walkable) — or wire the waypoint path.
- Enemy prefab: `add_navmesh_agent` + `Crawler` brain + `EnemyHealth`; reaching the goal does `BaseHealth.TakeDamage(1)` and despawns.
- `WaveSpawner` → enemy prefab; `WaveDirector.OnPhaseChanged` enables spawning during Wave.
- Towers: `PlacementController` charges `Wallet.TrySpend(cost)` then instantiates a `Tower`; the tower auto-targets + fires.
- Enemy `EnemyHealth.OnDeath` → `Wallet.Add(bounty)`.
- HUD bound to Wallet, BaseHealth (lives), and the wave phase.

## How to verify each step

- **Pathing:** `get_navmesh_path from="<spawn>" to="<goal>"` MUST return a walkable path — if not, re-`bake_navmesh` or fix the lane. This is the #1 thing that breaks the genre.
- **Tower fire:** `sim_play`, `teleport` an enemy into a tower's range, and confirm the enemy's `HealthFraction` drops over time (`get_runtime_property`) — that proves acquire+fire without waiting for a full wave.
- **Placement charge:** `invoke_method` the placement confirm at a spot, confirm a tower spawned (`find_in_play`) AND `Wallet` dropped (`get_runtime_property`); test the can't-afford reject.
- **Waves + lives:** `sim_set_speed speed=4`; enemies spawn, walk the lane, and reaching the goal drops `BaseHealth` (read it). `screenshot_game` to watch the lane; read the PNG.
- **Bounty:** kill an enemy (`invoke_method TakeDamage` to death) and confirm `Wallet` rose.

## Gotchas

- **No valid path = the whole game is broken** — always confirm `get_navmesh_path` after baking and after placing towers (towers shouldn't block the only lane unless that's intended — if they can, re-check pathability after placement).
- **Tower targeting needs colliders on enemies** to detect/hit — verify with `get_component_info`.
- **Charge on commit through `TrySpend`** — never let a tower be placed for free.
- **`create_round_phase_machine` self-ticks** — read the phase, don't double-advance it.
- **Cap concurrent enemies + prune the live list** so a stalled wave doesn't leak.

## Cross-links

Systems: spawning-waves, npc-ai, economy-currency, building-placement, health-combat, ui-hud. Shares the placement pattern with `references/genres/tycoon-management.md`. Workflow: `claude-bridge-build-feature`.
