---
name: claude-bridge-scaffold-game
description: Use when the user asks for a whole playable game in one go through the Claude Bridge for Unity — "make me a first-person game", "scaffold a game", "give me something I can press Play on". Orchestrates the bridge's gameplay scaffolds (create_player_controller / create_objective_system / create_game_manager / create_health_system / create_pickup / create_trigger_zone + the primitives, colliders, camera, and set_component_reference wiring) into a first-person starter you can enter Play mode in and actually move around, see a level, and win or lose. Handles the generate→recompile→place sequencing and the screenshot verify loop.
---

# Scaffold a Playable Unity Game

This skill turns ONE ask ("make me a first-person game") into a scene the user can press
**Play** on and actually *move around, see a level, and win or lose* — not a folder of
`.cs` files they still have to wire up by hand.

**Phase 1 ships one genre: `first_person`.** It exercises every part of the scaffold
pipeline and the trickiest constraint (generate→recompile→place sequencing). The bridge's
`create_player_controller` also generates `third_person` and `top_down` controllers, so once
the first-person path is solid those are small variations — but if the user asks for them
now, say the *guided scaffold* is first-person for the moment and offer either the
first-person starter or a hand-built approach via `claude-bridge-build-feature`.

This skill is a **sibling of `claude-bridge-build-feature`** and inherits its disciplines:
**you have no eyes without a screenshot, and reflection is the source of truth, not your
training data.** Read that skill's gotcha table too. To write/extend the generated C#
correctly, `claude-bridge-api` is the brain.

---

## The one constraint that breaks scaffolds: two-phase sequencing

A freshly generated C# type is **not in a loaded assembly until Unity recompiles and the
domain reloads.** So you cannot generate a script and place that component in the same breath
— `component_add` will fail to resolve the type ("not found"). Every scaffold here runs in
two phases:

1. **Phase A — GENERATE all scripts**, then `editor_recompile` → `wait_ready` →
   `get_compile_errors` (confirm clean).
2. **Phase B — BUILD + PLACE + WIRE the scene** (create objects, `component_add` the
   now-loaded components, set fields, `set_component_reference` to wire live references).

Do not interleave them. The `create_*` scaffolds each say in their result that they wrote +
imported the script but did **not** attach it — that is your signal to finish Phase A, hotload,
and place in Phase B. If `get_compile_errors` is red after Phase A, fix the generated file
(`edit_script`) and re-recompile before touching the scene.

---

## Step 0 — Bridge alive, scene safe, input sane

```
mcp__claude-bridge-unity__bridge_status
```
If it times out: Unity isn't running, the package isn't installed, the proxy isn't wired into
your MCP client (restart the client after wiring — MCP loads at startup), or Unity is
mid-compile. Stop until it responds. `ping` is the lightest liveness check.

**Not in play mode** (scene-mutating tools refuse during play):
```
mcp__claude-bridge-unity__sim_status        # poll the real isPlaying
mcp__claude-bridge-unity__sim_stop          # if it reports playing, then poll sim_status until stopped
```

**Scene safety** — look before you mutate:
```
mcp__claude-bridge-unity__scene_get_info
mcp__claude-bridge-unity__scene_hierarchy   maxDepth=1
```
- Empty scene (or just a Camera + Directional Light) → proceed.
- The user's **existing work** is in it → ASK: build into a fresh scene (File ▸ New Scene,
  or have them open one) or into this one? Default to a fresh scene so nothing is clobbered.

**Input handling — the #1 silent-no-movement gotcha.** The generated controllers use the
**legacy Input manager** (`Input.GetAxis`/`GetButton`) for broad compatibility. If the
project's **Active Input Handling** (Project Settings ▸ Player) is **"Input System Package
(New)" only**, legacy `Input.*` calls throw at runtime and the player won't move. Tell the
user to set Active Input Handling to **"Both"** (or "Input Manager (Old)") before Play — or,
if they're committed to the new Input System, say the generated controller needs swapping to
it and hand that to `claude-bridge-build-feature`.

Confirm the genre. If the user was vague ("make me a game"), tell them the guided scaffold
builds a **first-person** starter and proceed unless they want otherwise.

---

## Phase A — Generate all scripts, recompile, verify clean

Generate everything you'll place BEFORE touching the scene. Minimal playable loop =
**move around a floored room and reach a goal to win, fall off and lose.**

1. **Player controller** (first-person, mouse-look + WASD + jump):
   ```
   mcp__claude-bridge-unity__create_player_controller   name="FpController"   mode="first_person"
   ```
   It writes `Assets/ClaudeBridge/Generated/FpController.cs` and imports it; it does NOT
   attach itself. (It drives a `CharacterController` — you'll add both in Phase B.)

2. **Game state brain** — a tiny win/lose state machine:
   ```
   mcp__claude-bridge-unity__create_game_manager   name="GameDirector"   statesCsv="Playing,Won,Lost"
   ```
   Generates a singleton `GameDirector` with a `GameState` enum + `ChangeState(GameState)` +
   `OnStateChanged`. (Prefer this over `create_objective_system` for the minimal win/lose
   loop — our objective system is a generic multi-objective *tracker*, great for quests but
   heavier than a single reach-the-goal needs.)

3. **A win-trigger glue script** — the bit that connects "player entered the goal" to "win".
   Generate a tiny MonoBehaviour with a serialized `GameDirector` reference that flips the
   state on trigger enter by the player:
   ```
   mcp__claude-bridge-unity__create_script   name="ReachGoal"   folder="Assets/ClaudeBridge/Generated"
   ```
   Then `edit_script` (or author it inline via `write_file`) so its body is:
   ```csharp
   using UnityEngine;
   public class ReachGoal : MonoBehaviour {
       [SerializeField] GameDirector director;   // wired in Phase B via set_component_reference
       void OnTriggerEnter(Collider other) {
           if (!other.CompareTag("Player")) return;
           if (director != null) director.ChangeState(GameDirector.GameState.Won);
           Debug.Log("Reached the goal — you win!");
       }
   }
   ```
   (Match the generated `GameDirector`'s actual enum/method names — read them back from the
   generated file or `describe_type GameDirector` after the recompile if unsure.)

4. *(Optional)* a hazard or collectibles — skip for the minimal starter:
   ```
   mcp__claude-bridge-unity__create_health_system   name="Health"   maxHealth=100
   mcp__claude-bridge-unity__create_pickup          name="Coin"   kind="coin"
   ```

Now recompile and CHECK before placing anything:
```
mcp__claude-bridge-unity__editor_recompile
mcp__claude-bridge-unity__wait_ready
mcp__claude-bridge-unity__get_compile_errors
```
`get_compile_errors` is honest (file:line). If a real error mentions one of YOUR generated
files, fix it and re-recompile. **Do not start Phase B until the compile is clean** — placing
a component whose type failed to compile will fail.

---

## Phase B — Build, place, and wire the scene

The generated types are now loaded. Build the playable scene.

### B1. Floor + walls (so you don't fall forever / the room reads as a room)
```
mcp__claude-bridge-unity__game_object_create_primitive   type="Plane"   name="Floor"
mcp__claude-bridge-unity__transform_set_scale            target="Floor"   value="5,1,5"
```
Add four walls as thin, tall `Cube` primitives at the room edges (a Plane is 10×10 units at
scale 1, so a 5× plane is 50×50 — place walls at ±25). Keep it simple; the point is an
enclosed space that screenshots clearly. A Plane already has a `MeshCollider`; Cubes already
have `BoxCollider`s — no extra colliders needed for static geometry.

### B2. Player + CharacterController + controller + eye camera
```
mcp__claude-bridge-unity__game_object_create     name="Player"
mcp__claude-bridge-unity__component_add          target="Player"   type="CharacterController"
mcp__claude-bridge-unity__component_add          target="Player"   type="FpController"
mcp__claude-bridge-unity__game_object_set_tag    target="Player"   tag="Player"      # built-in tag
mcp__claude-bridge-unity__transform_set_position target="Player"   value="0,2,0"     # above the floor
```
Child a camera at eye height:
```
mcp__claude-bridge-unity__game_object_create     name="Eye"   parent="Player"
mcp__claude-bridge-unity__component_add          target="Player/Eye"   type="Camera"
mcp__claude-bridge-unity__game_object_set_tag    target="Player/Eye"   tag="MainCamera"
mcp__claude-bridge-unity__transform_set_position target="Player/Eye"   value="0,1.6,0"   # local; verify it resolves local
```
Then **read the controller's serialized fields** and wire anything it needs (some generated
first-person controllers expose a camera/pitch-pivot field):
```
mcp__claude-bridge-unity__get_component_fields   target="Player"   component="FpController"
# if it has e.g. a 'cameraPivot'/'lookCamera' field:
mcp__claude-bridge-unity__set_component_reference target="Player" component="FpController" field="<thatField>" sourceObject="Player/Eye"
```
> If the project's stock `Main Camera` is still in the scene, **delete it or remove its
> MainCamera tag** — two MainCamera-tagged cameras fight and `screenshot_game`/`Camera.main`
> picks the wrong one. (`scene_validate` flags this.)

### B3. The GameDirector singleton
```
mcp__claude-bridge-unity__game_object_create   name="GameDirector"
mcp__claude-bridge-unity__component_add        target="GameDirector"   type="GameDirector"
```

### B4. The goal (win trigger) at the far end
```
mcp__claude-bridge-unity__game_object_create_primitive   type="Cube"   name="Goal"
mcp__claude-bridge-unity__transform_set_position         target="Goal"   value="0,1,40"
mcp__claude-bridge-unity__transform_set_scale            target="Goal"   value="2,2,2"
# make its existing BoxCollider a trigger, and add the glue:
mcp__claude-bridge-unity__component_set        target="Goal"   component="BoxCollider"   field="isTrigger"   value="true"
mcp__claude-bridge-unity__component_add        target="Goal"   type="ReachGoal"
# WIRE the glue's director reference to the live GameDirector object:
mcp__claude-bridge-unity__set_component_reference   target="Goal"   component="ReachGoal"   field="director"   sourceObject="GameDirector"
```
Make it read as a green goal pad (so the screenshot is unambiguous) — assign a green material
via the material tools, or `create_asset assetType="Material"` + assign. `set_component_reference`
is the tool that wires a LIVE scene object (or a project asset) into a serialized reference
field — the thing flat-scalar `component_set` can't express. **Verify the wire:**
```
mcp__claude-bridge-unity__get_component_fields   target="Goal"   component="ReachGoal"   # director should resolve to GameDirector
```
> If `set_component_reference` isn't in your installed package version, fall back to
> `execute_csharp` to assign the reference, or ask the user to drag GameDirector onto the
> ReachGoal `director` slot in the Inspector.

### B5. Minimal HUD (optional but nice)
```
mcp__claude-bridge-unity__create_canvas        name="HUD"
mcp__claude-bridge-unity__ensure_event_system
mcp__claude-bridge-unity__create_ui_text       parent="HUD"   text="Reach the green pad to win"
```

### B6. A lose condition (optional, Phase 1-friendly)
Falling off the edge = lose. The simplest robust version: add a large trigger Cube well below
the floor wired to a second tiny glue (or reuse `ReachGoal`'s pattern flipping to `Lost`). Or
have `GameDirector` watch `transform.position.y < killY`. Keep it minimal; offer it as a
follow-up rather than blocking the first playable.

---

## Step 2 — Save, then VERIFY with your own eyes

```
mcp__claude-bridge-unity__editor_save_scene
```

**Structural check** — confirm the objects/components exist and references resolved:
```
mcp__claude-bridge-unity__scene_hierarchy   maxDepth=2
mcp__claude-bridge-unity__scene_validate          # catches null materials (magenta), dup MainCamera, multiple AudioListeners, missing camera
```
You should see: Floor (+walls), Player (CharacterController + FpController) with a child Eye
(Camera, MainCamera), GameDirector, Goal (BoxCollider trigger + ReachGoal → director wired),
HUD.

**Visual check** — aim the camera and READ THE PNG (you're multimodal):
```
mcp__claude-bridge-unity__screenshot_orbit   target="Player"   distance=6     # player sits ON the floor
mcp__claude-bridge-unity__screenshot_orbit   target="Goal"     distance=8     # green goal visible at the far end
```
**Look at the images.** If the player is floating/sunk, the room isn't enclosed, or the goal
is missing/mis-coloured, fix the transforms/materials and re-shoot. Don't declare it done from
the code looking right — the screenshot loop closes faster than the guess loop.

**Compile check** (belt and suspenders after the wiring):
```
mcp__claude-bridge-unity__get_compile_errors
```

---

## Step 3 — Hand it back in plain language

Tell the user, in non-coder terms, exactly what was built and **how to play it**:

> Built a first-person starter you can play now:
> - A floored room with walls, a player you control, and a green goal pad at the far end.
> - **Press Play, move with WASD, look with the mouse, jump with Space, and reach the green
>   pad to win.**
> - The win logic is in `GameDirector` + a small `ReachGoal` trigger; your movement is in
>   `FpController.cs` — all plain, commented C# you can read and tweak.
> - One setup note: movement uses Unity's classic Input manager, so make sure **Project
>   Settings ▸ Player ▸ Active Input Handling** is **"Both"** (or the old manager).

Then state the **human verification step** honestly: edit-mode screenshots prove the scene is
laid out right, but **they cannot prove the movement feels right or the win actually fires —
that's runtime feel, which the bridge can't synthesize.** Ask the user to press Play and
confirm they can move and reach the goal, and to tell you what feels off (jump height, mouse
sensitivity, speed) so you can tune it. You *can* enter Play yourself (`sim_play` → poll
`sim_status`), read `editor_read_log` for runtime errors, and `screenshot_game` a frame — do
that to catch hard errors, but defer "feel" to the human.

---

## Adapt, don't hardcode

- **Theme/name:** if the user gave a theme, name the scene/objects accordingly and pick
  fitting materials/primitives; the structure is identical.
- **Reuse what's installed:** if the project already has a controller (Starter Assets, an
  asset-store FPS kit), prefer adding *its* component over generating one — confirm its type +
  field names with `describe_type` first. State which path you took.
- **Don't gold-plate:** a playable reach-the-goal room beats a half-wired epic. Ship the
  minimal playable loop, then offer extensions (pickups via `create_pickup`, a hazard via
  `create_health_system`, a quest via `create_objective_system`, day/night via
  `create_day_night_clock`) as follow-ups using the same tools.

## When something half-works

The usual failure is the two-phase ordering. If `component_add` returns "type not found" or a
scaffold's result said the type isn't attached yet: you skipped or raced the recompile. Re-run
`editor_recompile` → `wait_ready` → `get_compile_errors` (confirm clean) → retry the placement.
If a reference won't set, `get_component_fields` to confirm the field name + that its type is a
GameObject/Component/asset (`set_component_reference` wires object/asset references; use
`component_set` for primitive values like floats/bools/strings). If the player won't move in
Play, it's almost always the **Active Input Handling** gotcha from Step 0.
