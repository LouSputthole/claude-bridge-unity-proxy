# Horror — Genre Recipe

Atmosphere first (it *is* the genre), then a stalker that hunts you, fragile vitals, and interact-to-progress objectives. Built from `apply_atmosphere`, the npc-brain (`predator`), health, and interactable scaffolds.

## The loop

Move through a dark, oppressive space → manage fragile resources (health, a flashlight, sanity) → an escalating threat (a stalker, the dark itself) hunts you → interact with the world to progress (find keys, fix things, escape) → survive or die. Horror is **pacing of pressure** + atmosphere; the mechanics are deliberately thin.

## The system stack to compose

1. **Atmosphere** — `apply_atmosphere mood="horror-night"` (composes ambient + fog + sun + skybox), `set_fog`, `add_light` for flickering/spot lights, `apply_post_fx_look look="noir"`. This is the cheapest, highest-leverage layer — **build it first**.
2. **Controller + flashlight** — `create_player_controller mode="first_person"` + a spot `add_light` parented to the camera.
3. **Vitals** — `create_health_system` (HP), optionally a second pool for a flashlight battery or sanity (`references/systems/health-combat.md`).
4. **The stalker** — `create_npc_brain preset="predator"` + `bake_navmesh` (`references/systems/npc-ai.md`).
5. **Interactables / objectives** — `create_interactable` (doors, keys, fixes) + `create_objective_system` (`references/systems/objectives-quests.md`).
6. **Audio** — ambience + stings (`references/systems/audio.md`).

## Bridge build order

**1. Atmosphere + blockout (do this FIRST and screenshot it).** Build an interior (boxes/corridors), then `apply_atmosphere mood="horror-night"`, `set_fog enabled=true density=0.05`, `apply_post_fx_look look="noir"`. `screenshot_scene` / `screenshot_game` and **read the PNG** — a correctly-lit empty maze is already scary; iterate the mood before any mechanics. A flickering light = a small `create_script` FSM toggling a Light's enabled state on random intervals.

**2. Phase A — generate, then compile:**
- `create_player_controller name="Survivor" mode="first_person"`
- `create_health_system name="Health" maxHealth=100` (+ a `Battery`/`Sanity` pool if used)
- `create_npc_brain name="Stalker" preset="predator"`
- `create_interactable name="KeyPickup" prompt="Press E to take" range=2.5` (+ a door interactable)
- `create_objective_system name="Objectives"`
- `editor_recompile` → `wait_ready` → `get_compile_errors`.

**3. Phase B — bake nav, place, wire:**
- Survivor: body + `component_add type="Survivor"` + `type="Health"`; parent a spot `add_light` to the camera for the flashlight; `add_audio_listener` on the camera.
- **`bake_navmesh`** (confirm `get_navmesh_path`); the stalker prefab gets `add_navmesh_agent` + `Stalker` brain; on catch → `Health.TakeDamage`.
- Interactables advance `Objectives` (take key → unlock door → escape goal).
- Ambient `add_audio_source loop=true`; a sting `play_sound` on the stalker spotting you.

## How to verify each step

- **Atmosphere (the most important verify):** `screenshot_game` / `screenshot_scene` and **read the PNG** — is it dark, foggy, oppressive? This is judged by eye; iterate `apply_atmosphere`/`set_fog`/`apply_post_fx_look` until it reads as horror. The flicker: confirm the Light's `enabled` toggles via `get_runtime_property`.
- **Stalker:** `bake_navmesh` + `get_navmesh_path` proves it can reach you; `sim_play`, `teleport` the player into line of sight, `simulate_npc_perception npcTarget="Stalker" targetObject="Survivor"` (edit-mode) or read the brain's state flipping to Chase; `screenshot_game` to see it close in.
- **Vitals:** `invoke_method Health.TakeDamage` → read `HealthFraction`; drain the battery and confirm the flashlight Light disables.
- **Interact → objective:** `teleport` to a key / `invoke_method` the interactable's `Interact`, confirm the objective advanced (`get_runtime_property`).
- **Feel/scares are human** — pacing of when the stalker appears is a playtest call; wire it, then iterate with the user.

## Gotchas

- **No NavMesh = the stalker can't hunt** — bake first; that single check makes or breaks the dread.
- **Atmosphere tools are Built-in RP** — `apply_atmosphere`/`set_fog`/`apply_post_fx_look` apply correctly here; don't expect URP/HDRP volumes.
- **A flicker FSM is local visual code** — it won't "fail" silently, but verify the Light toggles via `get_runtime_property`, not by assuming.
- **Interact range too small = the player can never trigger it** — confirm `range` with `get_component_fields`.
- **Two-phase** for every scaffold; the bridge can't judge whether a scare lands — that's the human's call.

## Cross-links

Systems: npc-ai, health-combat, objectives-quests, audio, day-night (`apply_atmosphere mood="horror-night"`). Close cousin: `references/genres/survival.md` (more gathering, less pure dread). Workflow + atmosphere screenshots: `claude-bridge-build-feature`.
