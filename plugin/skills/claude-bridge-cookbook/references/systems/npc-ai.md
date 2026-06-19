# NPC / Enemy AI (FSM + NavMesh)

A perception-driven state machine for movement + behavior, navigating on Unity's built-in NavMesh. Built on the real `create_npc_brain` scaffold plus the NavMesh tools.

## What it IS / when you need it

An NPC "brain" MonoBehaviour running a small FSM (Idle/Patrol/Chase/Search/Flee) driven by simple perception, moving via a `NavMeshAgent`. Use it for guards, wandering townsfolk, predators that hunt the player, or cowards that flee. The brain decides *what*; the NavMeshAgent does the *moving*.

## Bridge build order

**Phase A — generate + compile:**
1. `create_npc_brain name="GuardBrain" preset="guard"` — generates the FSM brain. Presets: `guard` / `wanderer` / `predator` / `coward`.
2. (Optional) `create_npc_spawner name="GuardSpawner" count=5 radius=12` if you want a spawner that instantiates the NPC prefab at randomized points capped by `maxAlive`.
3. `editor_recompile` → `wait_ready` → `get_compile_errors`.

**Phase B — bake nav + place + wire:**
4. **Bake the NavMesh first** — the agent can't path without it. Lay out the level (ground + obstacles), then `bake_navmesh`. Confirm a route exists: `get_navmesh_path from="<A>" to="<B>"` returns a path (or `reachable:false`, which tells you the bake missed or the points are off-mesh).
5. Build the NPC: a body (`game_object_create_primitive` or `prefab_instantiate`), then `add_navmesh_agent target="Guard" speed=3.5 radius=0.5`, then `component_add target="Guard" type="GuardBrain"`.
6. **Patrol route:** `place_patrol_route name="Route1" pointsJson="[\"0,0,0\",\"10,0,0\",\"10,0,10\"]"` creates an empty parent with ordered waypoint children; `assign_patrol_route npcTarget="Guard" routeName="Route1"` wires them into the brain.
7. Perception target is usually the player — set whatever target field the brain exposes (`component_set`).

## Verify

- **NavMesh sanity (before play):** `get_navmesh_path` between the NPC and a destination must return a walkable path. If it can't, re-`bake_navmesh` (check the geometry is marked navigation-static / walkable).
- **Perception without play mode:** `simulate_npc_perception npcTarget="Guard" targetObject="Player"` runs the line-of-sight / FOV / range check *now*, in edit mode — a read-only verifier that tells you whether the guard can currently "see" the player. Move the player (`transform_set_position`) and re-run to find the FOV edges.
- **Runtime:** `sim_play` → poll `sim_status`. `teleport target="Player" position="<near the guard>"` and read the brain's state (`get_runtime_property target="Guard" componentType="GuardBrain" propertyPath="<state field>"`) — it should flip from Patrol to Chase. `frame_object target="Guard"` + `screenshot_game` to watch it move; read the PNG.

## Variations

- **Presets map to archetypes:** `guard` patrols then chases on sight; `wanderer` roams; `predator` actively hunts; `coward` flees. Pick the closest, then tune.
- **Attacking:** the brain handles movement/targeting; pair it with `create_health_system` on the player and a small "attack when in range" call (`edit_script` the brain or a sibling component) that does `TakeDamage`.
- **Crowds (non-combat):** for many wandering bodies, prefer one spawner + cheap brains; don't put heavy `Update` logic on hundreds of agents.
- **Spawned enemies:** combine with `references/systems/spawning-waves.md` (`create_spawner` / `create_npc_spawner`).

## Gotchas

- **No NavMesh = no movement.** `add_navmesh_agent` on an object with no baked mesh under it just sits there. **`bake_navmesh` before you expect motion**, and re-bake after you change the level geometry.
- **`NavMeshAgent` member names drift** — verify with `describe_type` (the AI/Navigation package API varies by version) before writing code against it.
- **Two-phase:** `create_npc_brain` → recompile → THEN `component_add`.
- **Agent radius vs door/gap width:** if a guard won't path through a gap, its `radius` is too big for the baked mesh — lower it (`add_navmesh_agent` reconfigures) or widen the geometry and re-bake.
- **Perception needs line-of-sight colliders** to occlude — `simulate_npc_perception` respects them, so a guard "seeing" through a wall means the wall has no collider.

## Cross-links

`references/systems/spawning-waves.md`, `references/systems/health-combat.md`, `references/systems/dialogue.md` (friendly NPCs you talk to). Genres: `references/genres/fps-shooter.md`, `references/genres/horror.md`, `references/genres/tower-defense.md` (enemies path to a goal).
