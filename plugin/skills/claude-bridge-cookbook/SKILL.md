---
name: claude-bridge-cookbook
description: Use when building a whole Unity game or a specific game SYSTEM through the Claude Bridge and you want a proven build order grounded in THIS bridge's real tools instead of guessing. A ROUTER over genre playbooks (tycoon/management, survival, fps-shooter, platformer, top-down twin-stick, tower-defense, rpg, horror, puzzle, racing) and per-system how-tos (inventory, save-load, economy-currency, health-combat, npc-ai, spawning-waves, progression-upgrades, dialogue, day-night, objectives-quests, ui-hud, audio, building-placement). Each reference names the exact bridge scaffolds/tools to compose (create_health_system, create_inventory, create_economy_wallet, create_npc_brain, create_save_system, create_player_controller, create_spawner, scatter_props, apply_atmosphere, screenshot_orbit, the editor_recompile→wait_ready→get_compile_errors trio, sim_play, set_runtime_property, invoke_method, …) and how to verify each step. Triggers on "how do I build a <genre> game", "inventory system", "save system", "economy / currency", "health / damage", "enemy AI", "spawn waves", "upgrade tree", "dialogue", "day/night", "quest / objective", "HUD", "tower defense", "twin-stick", "building placement", and similar. This is a ROUTER — find the system/genre, open the reference; do not answer from SKILL.md alone.
---

# Claude Bridge — Unity Cookbook (Master Router)

A library of **build orders** grounded in the **real Claude-Bridge-for-Unity tool surface** (see `docs/TOOLS.md` for the authoritative, always-current catalog). Each recipe tells you *what bridge scaffolds/tools to compose and in what order*, how to set up the scene, and how to **verify each step** so you never declare a thing done you haven't seen.

**This file is an index, not the answer.** Find your system or genre below, then **open that reference** so you load only what you need.

## The five skills (+ one agent), and when each fires
- **`claude-bridge-cookbook`** (this) — *recipes*: "how do I build a **tycoon** / an **inventory** / a **save system**?" → routes to a grounded build order.
- **`claude-bridge-scaffold-game`** — *one-shot starter*: "make me a **first-person game** / something I can press Play on" → orchestrates the scaffolds into a playable reach-the-goal starter (the whole-game recipes below are for deeper/custom builds; this is the fast path to *playable*).
- **`claude-bridge-api`** — *the brain*: how to write correct Unity C# (MonoBehaviour lifecycle, `[SerializeField]` vs public, Editor vs runtime, the s&box/Unreal→Unity table, serialization rules). Open it for the exact pattern.
- **`claude-bridge-build-feature`** — *hands + eyes*: the screenshot-driven bridge workflow, the **two-phase rule**, play-mode gotchas. Open it to build it, run it, and SEE it. **Every recipe below assumes that workflow** — it is not repeated in each file.
- **`claude-bridge-setup`** — first-run onboarding + connection verify.
- **`unity-game-dev`** (agent) — hand a self-contained build off to a specialist subagent that runs the whole build-feature loop itself.

**Authority of truth:** these recipes teach the pattern and name real tools; the **live editor reflection** (`describe_type` / `search_types` / `get_method_signature` / `list_components`) is the authoritative signature check for *your* Unity version + installed packages. If a recipe disagrees with live reflection, reflection wins.

> **Scaffold reality.** The high-level scaffolds named below (`create_health_system`, `create_economy_wallet`, `create_inventory`, `create_save_system`, `create_player_controller`, `create_game_manager`, `create_spawner`, `create_npc_brain`, `create_round_phase_machine`, `create_day_night_clock`, `create_objective_system`, `create_interactable`, `create_pickup`, `create_trigger_zone`, `create_stat_modifier_system`, `create_weighted_loot_table`, `create_objective_marker`) each **generate a complete, self-compiling C# file** — they are codegen, not runtime wiring. A generated type is **not loadable until a recompile + domain reload finishes** (the two-phase rule). After any scaffold: `editor_recompile` → `wait_ready` → `get_compile_errors` (clean) **before** you `component_add` it or set its fields. Where no scaffold fits, write the C# by hand with `create_script` / `write_file` and compile-verify identically.

---

## 🎮 Building a whole game? → Genre recipes
> **Want it playable in one shot first?** If the user just wants "a game I can press Play on,"
> run **`claude-bridge-scaffold-game`** for an end-to-end first-person starter, then use these
> genre recipes to deepen it. The recipes below are the *custom/deeper* path.

Each `references/genres/<x>.md` gives **the loop**, the **system stack to compose**, a **bridge build order** (which scaffolds/tools in what sequence), the **scene setup**, and **how to verify each step**.

| You're building… | Open | Spine scaffolds |
|---|---|---|
| Tycoon / management / idle | `references/genres/tycoon-management.md` | `create_economy_wallet`, `create_save_system`, `create_interactable`, `create_day_night_clock` |
| Survival (vitals, gather, craft) | `references/genres/survival.md` | `create_health_system`, `create_inventory`, `create_pickup`, `create_spawner`, `create_save_system` |
| FPS / shooter | `references/genres/fps-shooter.md` | `create_player_controller (first_person)`, `create_health_system`, `create_npc_brain`, `create_spawner` |
| Platformer / obstacle | `references/genres/platformer.md` | `create_player_controller (third_person)`, `create_trigger_zone`, `create_objective_system` |
| Top-down / twin-stick shooter | `references/genres/top-down-twin-stick.md` | `create_player_controller (top_down)`, `create_spawner`, `create_health_system`, `create_weighted_loot_table` |
| Tower defense | `references/genres/tower-defense.md` | `create_spawner`, `bake_navmesh`, `create_economy_wallet`, `create_health_system`, `create_round_phase_machine` |
| RPG (stats, quests, loot) | `references/genres/rpg.md` | `create_stat_modifier_system`, `create_inventory`, `create_objective_system`, `create_weighted_loot_table`, `create_save_system` |
| Horror (atmosphere, AI, scares) | `references/genres/horror.md` | `apply_atmosphere`, `create_npc_brain (predator)`, `create_health_system`, `create_interactable` |
| Puzzle (rooms, triggers, gates) | `references/genres/puzzle.md` | `create_trigger_zone`, `create_interactable`, `create_objective_system`, `create_game_manager` |
| Racing / driving | `references/genres/racing.md` | `create_player_controller`, `physics_add_rigidbody`, `create_trigger_zone`, `create_round_phase_machine` |

For every genre: **block out the scene first** (a ground plane + primitives via `game_object_create_primitive`, screenshot to confirm framing), THEN generate scripts and compile-verify, THEN place + wire, and finally `sim_play` and read a `screenshot_game`. That whole arc is the `claude-bridge-build-feature` loop.

## 🧩 Need one system? → System how-tos
Each `references/systems/<x>.md` = what it is + the canonical Unity approach + which bridge scaffold/tools to use + how to verify + gotchas.

| System | Open |
|---|---|
| Inventory / hotbar / pickup | `references/systems/inventory.md` |
| Save / load / persistence | `references/systems/save-load.md` |
| Economy / currency / wallet | `references/systems/economy-currency.md` |
| Health / damage / combat | `references/systems/health-combat.md` |
| NPC / enemy AI (FSM + NavMesh) | `references/systems/npc-ai.md` |
| Spawning / waves | `references/systems/spawning-waves.md` |
| Progression / upgrades / stats | `references/systems/progression-upgrades.md` |
| Dialogue / conversation | `references/systems/dialogue.md` |
| Day / night cycle | `references/systems/day-night.md` |
| Objectives / quests | `references/systems/objectives-quests.md` |
| UI / HUD (uGUI) | `references/systems/ui-hud.md` |
| Audio (SFX, music, ambience) | `references/systems/audio.md` |
| Building / placement | `references/systems/building-placement.md` |

---

## The cross-cutting laws (true in every recipe)

These bite across every system — internalize them; the references repeat them but they live here so you don't have to open one to remember them:

1. **Two-phase always.** A generated C# type isn't loadable until `editor_recompile` + the domain reload finish. **Phase A:** generate every script → `editor_recompile` → `wait_ready` → `get_compile_errors` (clean). **Phase B:** create objects, `component_add` the now-loaded types, set fields, wire references, parent. Never interleave — `component_add` on an uncompiled type fails to resolve.
2. **Flat params, always.** Vectors/colors are **comma strings**, never JSON objects: position `"0,1.5,0"`, scale `"2"` (uniform) or `"1,2,1"`, rotation euler degrees `"0,90,0"`, color `"r,g,b,a"` or `#RRGGBB`. This is the whole param convention.
3. **Verify the mutation.** The bridge tools echo the real value — **read it back**. If `transform_set_scale` returns a `localScale` that isn't what you asked, it no-op'd. "I called the tool" ≠ "it took." Confirm the resolved `path`/`instanceId` is the object you meant (names aren't unique — prefer the `A/B/C` hierarchy path or instanceId).
4. **See visual changes.** Aim a camera at the subject (`screenshot_orbit target=… distance=…`, or `frame_object` then `screenshot_scene`) and **read the PNG yourself** — you're multimodal. A raw `screenshot_game` only helps if the active camera already frames the subject. The screenshot loop closes faster than the guess loop.
5. **Compile-verify generated code.** After any `.cs` write or scaffold: `editor_recompile` → `wait_ready` → `get_compile_errors`. `get_compile_errors` is honest (file:line). Don't proceed while red.
6. **Reflection-first.** Unity's API differs across versions, render pipelines (this bridge is **Built-in RP**), and packages. Before calling a type/member you haven't confirmed, `describe_type` / `get_method_signature` / `list_components`. Don't assume a name from training data (`velocity`→`linearVelocity`, `FindObjectsOfType`→`FindObjectsByType`). To see what's actually on a live object: `get_component_info` (the list) + `get_component_fields` (the values).
7. **Serialize correctly.** `[SerializeField]` for inspector fields; only Unity-serializable types persist; **no `Dictionary` / interface in serialized state** — use a `[System.Serializable]` list of structs. (Detail in `claude-bridge-api`.)
8. **Physics in `FixedUpdate`, input/logic in `Update`,** rates scaled by `Time.deltaTime` / `Time.fixedDeltaTime`. Move physics bodies via the Rigidbody, not raw transform sets, or you fight the solver.
9. **Persisted state must version + clamp on load.** Saves outlive your balance changes — `create_save_system` gives you the seam; sanitize every field on read. Player saves go to `Application.persistentDataPath`, **never** `AssetDatabase` (editor-only; guard editor calls with `#if UNITY_EDITOR`).
10. **Feel is a human call, and the bridge drives ONE editor.** The bridge can't press keys or judge whether the jump feels right — hand movement/camera/difficulty tuning to the user to playtest (it CAN drive runtime state via `set_runtime_property` / `invoke_method` / `teleport` and best-effort `simulate_input` / `press_and_hold`). And parallel agents may AUTHOR disjoint files only — one orchestrator recompiles, enters play, and screenshots serially.

---

*Router only. Find the system or genre, open its reference, build it with `claude-bridge-api`, and SEE it with `claude-bridge-build-feature`. Verify exact API live. Every tool named in a reference is real — cross-check `docs/TOOLS.md`.*
