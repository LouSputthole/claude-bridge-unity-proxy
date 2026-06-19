---
name: claude-bridge-build-feature
description: Use when building, modifying, or polishing any feature in a Unity project through the Claude Bridge — gameplay systems, components, prefabs, scene layout, materials/lighting, UI, anything that produces a visible or runtime change. Codifies the screenshot-driven iteration workflow that prevents the "guess-and-check" loop the bridge is most susceptible to, plus the hard-won Unity gotchas to bake in from day one.
---

# Building Unity Features Through the Bridge

This skill is the workflow you follow whenever you're about to make non-trivial changes to a Unity project via the Claude Bridge. It exists because the bridge gives Claude a lot of power but **no eyes** — without discipline, sessions devolve into guessing about how things look and whether a change actually took. These steps prevent that.

**Pair this with the `claude-bridge-api` skill** — that's the *brain* (how to write correct Unity C#: MonoBehaviour lifecycle, `[SerializeField]` vs public, Editor vs runtime, `AssetDatabase`, serialization rules). This `claude-bridge-build-feature` skill is the *hands + eyes* (drive the editor, screenshot, verify live). Write it right with `claude-bridge-api`; then build it, run it, and SEE it with the bridge. And the bridge's live reflection (`describe_type` / `search_types` / `get_method_signature`) is the authoritative signature check for *your* installed Unity + packages — better than training data.

## Hard rule: never declare a change "done" without verifying it

Two failure modes the bridge is prone to, and the discipline for each:

1. **Visual changes** (a transform, a primitive, a material, a light, a prefab placement, UI): you **must see it** before saying the work is done. You're a multimodal model — read the PNG. Aim a screenshot at the thing you changed, capture, and look.
2. **State mutations** (set a position, add a component, set a field, rename, reparent): you **must confirm the mutation took**. The bridge tools are written to verify and return the real value — read it back. "I called the tool" is not "it worked." (Some Unity setters silently no-op or get overwritten by serialization — see the gotchas table.)

## The Workflow — six steps, in order

### 1. Confirm the bridge is alive

```
mcp__claude-bridge-unity__bridge_status
```

A healthy reply has `bridgeVersion`, `toolCount`, `unityVersion`, `projectName`, `isPlaying`. If it times out: Unity isn't running, the proxy isn't wired in (MCP loads at Claude Code startup — a restart is required after wiring), or Unity is mid-compile/mid-domain-reload. The proxy buffers calls across a reload, so a normal recompile shouldn't surface an error — but don't start building until status responds. `ping` is the lightest liveness check.

### 2. Brainstorm before code (for non-trivial features)

If the feature is more than a one-line tweak — anything that involves:
- A new state machine or system
- A new MonoBehaviour or ScriptableObject that other code depends on
- Animation, physics tuning, camera work, or anything where you can't predict the visual/runtime outcome with confidence

…invoke `superpowers:brainstorming` first. Don't skip this. The cost of designing wrong is much higher than the cost of designing slowly.

### 3. Research the Unity API before guessing (reflection-first)

Before writing code that calls a type or method you haven't verified exists in the current Unity version + installed packages:

```
mcp__claude-bridge-unity__describe_type        typeName="Rigidbody"
mcp__claude-bridge-unity__search_types         query="NavMesh"
mcp__claude-bridge-unity__list_components      query="Collider"
mcp__claude-bridge-unity__get_method_signature typeName="GameObject" methodName="AddComponent"
```

Unity's API differs across versions, render pipelines (Built-in vs URP vs HDRP), and packages. **Reflection is the source of truth, not your training data.** `describe_type` resolves a type across all loaded assemblies (Unity modules, your `Assembly-CSharp`, package asms) and lists its fields/properties/methods. If it doesn't show a member, you're guessing — stop and look. To inspect what's actually on a live object, `get_component_info` (the component list) and `get_component_fields` (the live VALUES) are your eyes.

For broader docs (a package's manual, render-pipeline setup, the input system), use WebFetch on the Unity Manual / Scripting API, or the `context7` MCP for package docs.

### 4. Implement with bite-sized edits

- One change per `Edit` call. Don't batch unrelated edits.
- Keep changes scoped to one file at a time when possible.
- **Confine writes to the project.** Generated C# goes under `Assets/` (or a package you own). Sanitize any generated class name — a name with spaces/punctuation/keywords produces a file that won't compile.
- **A new C# type is not usable until Unity recompiles AND the domain reloads** (see step 5 + the two-phase rule below). Don't generate a MonoBehaviour and try to `component_add` it in the same breath — the type isn't in the loaded assembly yet.

### 5. Compile-verify generated code

This is the Unity equivalent of "did it build." After writing or editing any `.cs`:

```
mcp__claude-bridge-unity__editor_recompile        # ask Unity to recompile / refresh
mcp__claude-bridge-unity__wait_ready       # wait until compilation + domain reload finish
mcp__claude-bridge-unity__get_compile_errors      # the real errors, file:line
```

- **`get_compile_errors` is honest** — it reports the actual compiler output with file + line. Read it. If there's a real error mentioning YOUR file, fix it (`Edit`), re-`editor_recompile`, re-check. Don't proceed while red.
- **A domain reload drops and re-establishes the bridge socket.** That's expected and the proxy rides it out (buffers in-flight calls, re-reads the status file for a possibly-new port, reconnects). Use `wait_ready` so you act only once the reload is done — calling tools mid-reload is how you get confusing partial results.
- If `editor_read_log` is available, it's the broader diagnostic when an error is non-obvious.

### 6. Screenshot and read it yourself

For any visual change, **aim a camera at the thing you changed, capture, and look at the PNG**:

```
mcp__claude-bridge-unity__screenshot_orbit   target=<name/path/instanceId> distance=<meters>
# or, for a framed editor/scene-view shot:
mcp__claude-bridge-unity__frame_object       target=<...>     # frame it in the Scene view first
mcp__claude-bridge-unity__screenshot_scene                     # then capture the Scene view
```

`screenshot_orbit` points a camera at your target from a sensible distance and angle so the subject is actually in frame — this is the single highest-leverage habit in the whole workflow, the Unity counterpart to "aim the camera or you'll screenshot the wrong thing." A raw `screenshot_game` renders the active Camera's current view — fine if it already frames your subject, useless otherwise.

After the tool returns, **`Read` the PNG** (the result carries the image, or a path to it). **Look at the image.** If it doesn't match the design, iterate. Don't declare the feature done because the code looks right. The screenshot loop closes faster than the guess loop.

For diagnosing a compile/runtime failure you don't need the editor to respond visually — `get_compile_errors` / `editor_read_log` read Unity's state directly.

## The two-phase rule for generated components (read this twice)

A freshly generated C# type is **not in a loaded assembly until Unity recompiles and reloads the domain.** So you cannot generate a script and place that component in the same step — `component_add` will fail to resolve the type.

**Every scaffold runs in two phases:**

1. **Phase A — GENERATE all scripts**, then `editor_recompile` → `wait_ready` → `get_compile_errors` (confirm clean).
2. **Phase B — BUILD + PLACE + WIRE the scene** (create objects, `component_add` the now-loaded components, set fields/references, parent things).

Do not interleave them. If `get_compile_errors` is red after Phase A, fix and re-recompile before touching the scene. Placing a component whose type failed to compile will fail.

## Seeing & driving the RUNNING game (play mode)

The bridge can verify *gameplay*, not just the edit scene — but play-mode tools behave differently:

- **Enter/exit play mode is asynchronous.** `sim_play` / `sim_stop` request the transition; entering or leaving play mode triggers a **domain reload**, so `isPlaying` flips on the *next* editor tick, not synchronously. The tools return `status:ok` with a `requested` field and a "pending" note — **poll `sim_status`** to confirm the flip. `sim_pause` / `sim_resume` are real (Unity exposes `isPaused`) and verify synchronously. `sim_set_speed` drives `Time.timeScale` for slow-mo / fast-forward.
- **Prove runtime behavior with live state, not vibes.** Read a component's runtime field with `get_component_fields` (works in play mode — it's read-only), or set/get runtime state and invoke methods via the runtime tools when you need to poke a value or fire logic. Unambiguous, and it works even behind a fullscreen UI.
- **Runtime-only changes are discarded on stop.** Anything you change in play mode (positions, field values) reverts when `sim_stop` completes. Author durable changes in edit mode.
- **`screenshot_game` is the play-mode eyes** (the running camera's view). Read the PNG it returns.

## The one thing the bridge cannot do: synthesize input "feel"

**The bridge cannot press keys, move the mouse, or feel the game.** It can enter play mode, set state, invoke methods, and screenshot a frame — but it cannot tell you whether the jump feels floaty, whether the camera is nauseating, whether the controls are responsive, or whether the difficulty curve is fun. Those are *human* judgments that require a controller in hand.

So when a feature's correctness lives in **feel** — movement, camera, timing windows, juice, difficulty — **hand it to the human**:
> The logic is in and compiles, and the scene is laid out right (here's the screenshot). I can't feel the controls, though — press Play, move around, and tell me: is the jump height right, does the camera lag too much, does it respond fast enough? Tell me what's off and by how much and I'll tune it.

State this honestly rather than claiming a feel-dependent feature is "done." For timing-sensitive captures (a specific animation frame), coordinate: "press the action and say 'go' the instant you do" and fire the screenshot immediately — the round-trip lands roughly the right window.

## Multi-agent work: the bridge drives ONE editor

The bridge connects to a single running Unity Editor. **Multiple agents cannot drive play-mode / screenshots / scene mutation concurrently** — they'd fight over the same editor, the same play session, the same camera, and the shared scene/asset files. The pattern that works for a parallel build:

- **Parallel agents AUTHOR on disjoint files only** — each owns a non-overlapping set of `.cs`/asset files and writes them with the normal file tools (Edit/Write), **no play-mode, no scene edits, no screenshots.**
- **One orchestrator verifies serially** — after the authors finish, a single agent (or the human) recompiles, drives play mode, screenshots, and reads. Don't have two agents call `sim_play`/`screenshot_*` against the same editor.
- If two things genuinely must run at once and both need the editor, they need **separate Unity instances each with their own bridge** (each writes its own status file; pin one with `CLAUDE_BRIDGE_PROJECT`).

## Common Unity gotchas (so you don't re-discover them)

| Gotcha | What to do instead |
|---|---|
| **Vectors/colors are flat `"x,y,z"` strings**, never nested objects | `transform_set_position` wants `"0,1.5,0"`; scale accepts `"2"` (uniform) or `"1,2,1"`; rotation is euler degrees `"0,90,0"`; color is `"r,g,b,a"`. This is the whole param convention — flat scalars and comma strings, no JSON objects. |
| A change "succeeded" but didn't take | The tools verify and return the real value — **read the echoed value back**. If `transform_set_scale` returns a `localScale` that isn't what you asked, it no-op'd. Trust the read-back, not the call. |
| A new C# type isn't found by `component_add` | Two-phase rule: generate → `editor_recompile` → `wait_ready` → `get_compile_errors` (clean) → only THEN place. The type isn't in the loaded assembly until the domain reloads. |
| Setting a field "from the inspector" gets wiped on save | A `[SerializeField]`/serialized field is restored from the scene/prefab on load — set it through the bridge as a real value, and remember field initializers act as defaults that serialization can override. (See `claude-bridge-api`.) |
| Scene-mutating tools refuse during play mode | They return a clear error. Stop play (`sim_stop`, then poll `sim_status`), mutate in edit mode, re-enter play if needed. Runtime changes are discarded on stop anyway. |
| `sim_play`/`sim_stop` don't flip `isPlaying` immediately | Entering/leaving play mode = domain reload; the flag flips next tick. **Poll `sim_status`.** Don't assume the transition completed because the call returned. |
| Unity's "fake null" on destroyed objects | A destroyed `UnityEngine.Object` compares `== null` but the C# reference is non-null. Inspection reports it as `"null (Type)"`. Don't treat a serialized reference as alive without checking. |
| `screenshot_game` renders the active camera's current view | If it's not aimed at your subject you'll screenshot the wrong thing. Use `screenshot_orbit target=… distance=…` (aims at the object) or `frame_object` then `screenshot_scene`. |
| Tag/layer assignment fails | The tag/layer must already exist in **Tags & Layers**. `game_object_set_tag` errors cleanly on an undefined tag; `game_object_set_layer` accepts a name or a 0–31 index. Define it first if missing. |
| Resolving the wrong GameObject by name | Names aren't unique. Prefer the **hierarchy path `A/B/C`** or the **instanceId** to disambiguate (every mutating tool echoes the resolved `path` + `instanceId` — confirm it hit the one you meant). |
| Editing a `.cs` on disk but Unity doesn't pick it up | Unity recompiles on focus/refresh; through the bridge call `editor_recompile` (or `editor_refresh_assets`) explicitly, then `wait_ready`. |
| Render pipeline mismatch (material looks pink/wrong) | A material/shader built for one pipeline (Built-in/URP/HDRP) renders broken in another. `describe_type` the renderer/material and verify the project's pipeline before mass-assigning a material. |
| Runtime `GameObject.CreatePrimitive(...)` renders MAGENTA under URP/HDRP | `CreatePrimitive` (and a bare `AddComponent<MeshRenderer>`) ships Unity's Built-in **`Default-Material`** (Standard shader), which the SRP replaces with the magenta error shader. Assign a real pipeline material (`new Material(Shader.Find("Universal Render Pipeline/Lit"))`) before the object is shown. (Temp primitives you only read a mesh from and `Destroy` are exempt.) |
| A `MaterialPropertyBlock` tint "does nothing" / stays magenta | An MPB can't recolor the magenta **error** shader — it ignores every property. Give the renderer a real pipeline material FIRST, *then* the MPB applies. Also: URP/HDRP Lit's colour is **`_BaseColor`**, not the Built-in `_Color` — set both for cross-pipeline safety. |
| Built-in→URP migration: the batch converter leaves materials magenta | `UnityEditor.Rendering.Universal.Converters.RunInBatchMode(BuiltInToURP)` runs RenderSettings/AnimationClip but commonly **no-ops the Material upgrade**. Hand-roll the remap via `execute_csharp` over `FindAssets("t:Material")` (Standard→URP/Lit + custom vendor shaders). `Resources.FindObjectsOfTypeAll<Material>()` only sees LOADED materials (runtime-only prefab mats are missed in edit mode — inspect the prefab/FBX directly). |
| Attaching a held prop/accessory to a character's hand | Resolve the bone via the Humanoid API — `Animator.GetBoneTransform(HumanBodyBones.RightHand)` — NOT a hardcoded bone NAME (engine-specific names like `hand_R`/`Bip01` won't exist on an arbitrary rig; the attach silently falls back to the object origin). Parent the prop there, strip its colliders, set a small local grip pose. |
| Imported NPCs slide / T-pose AND spam "Parameter does not exist" | Their prefab ships a controller with no locomotion params. Bind a shared Humanoid locomotion controller to any Humanoid Animator NOT already on yours (not just controller-LESS ones), set `applyRootMotion=false`, and `SetFloat("MoveSpeed", planarSpeed, damp, dt)` from the transform's real motion each frame. A Humanoid clip (e.g. Mixamo, imported with `animationType=Human`) retargets onto any Humanoid avatar. |
| Bridge tools all 25s-time-out (incl. `ping`) | The editor's main thread is saturated (big asset reimport, a multi-GB `Editor.log`, a slow domain reload) — not a bridge fault. Back off; do disk-only work (it applies on the next recompile); re-probe `ping` after a pause rather than hammering. |
| Assuming `Debug.Log` output is visible to you | You don't see the Game/Console window. Use `get_compile_errors` / `editor_read_log` to read what Unity logged, or read live values with `get_component_fields`. |
| Guessing an enum literal or method overload | `get_method_signature` lists every overload with param names/types/defaults; `describe_type` shows enum membership. Look it up rather than guessing a literal that won't compile. |
| A bootstrap-created camera/light is doubled or silently overridden | If the scene asset ships a stock **Main Camera** or **Directional Light** and your bootstrap only adds one *"if none exists"*, BOTH go live: two cameras → a split/double view; a stock dark light wins so your warm sun never shows and everything renders as black silhouettes. Make the bootstrap AUTHORITATIVE — find the stock object and reconfigure/disable it, don't skip-if-exists. (Bit one project twice: camera double-vision, then a near-black "day".) |
| A scene "looks wrong" and you're tempted to swap assets | READ the live values first — the cause a screenshot hides is usually ONE component. `get_component_fields` on a sun showed `intensity 1.0` but `color 0.12,0.13,0.14` (near-black) → a LIGHTING bug, not 50 broken prefabs. `frame_object` in PLAY mode renders under the runtime lighting, so a still-dark close-up = lighting / a correct one = it was just unlit. A `Test-Path` (or `find_assets`) sweep of every referenced asset path proves whether anything is actually missing. |
| A screenshot misses your IMGUI / screen-space UI | `screenshot_game` renders the active Camera ONLY — not `OnGUI`/IMGUI or screen-space-overlay Canvas UI. Use `screenshot_game_view` for the **composited** Game View (camera + every UI overlay) when verifying HUDs, pause menus, on-screen text. |
| Need "the object that HAS component X" (a camera, light, manager) | `find_in_play componentType="Camera"` (any type, including your own game scripts) returns every active GameObject carrying it — runtime-spawned managers/lights rarely have predictable names, so search by TYPE, not name. Pair with `get_component_fields` / `get_runtime_property` to read its live state. |

## Project-level CLAUDE.md

If the project you're working on has its own `CLAUDE.md`, **read it first.** It captures project-specific decisions (input bindings, scene layout, naming conventions, render pipeline, package set) that this skill can't know about.

## The thing that always works

When you're stuck, in a loop, or about to make your fifth guess at a transform offset or a material tweak:

1. Screenshot the current state (aimed at the subject) and **read it yourself**.
2. Read the live values with `get_component_fields` so you know the *actual* numbers, not the ones you assumed.
3. Describe to the user exactly what you see vs. what should be there.
4. Propose a specific adjustment **with magnitude** ("move it +0.5 in Y", "drop the light intensity to 1.2") rather than another blind guess.

The screenshot-and-read-back loop closes faster than the guess loop. Use it.
