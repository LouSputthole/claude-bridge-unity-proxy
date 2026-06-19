---
name: claude-bridge-live-patch
description: Use when iterating on GAMEPLAY LOGIC in play mode through the Claude Bridge — tuning a method body and wanting to see the effect WITHOUT the domain-reload tax (which exits play and drops runtime state). Covers the Live Patch family (live_patch_apply / live_patch_status / live_patch_revert / live_patch_list): when it applies a method-body change to the running domain instantly, when it must fall back to a normal recompile, and how to fold it into the screenshot-driven playtest loop. EXPERIMENTAL; requires the optional com.lousputthole.livepatch package.
---

# Iterating with Live Patch

Live Patch applies a changed C# **method body** to the **already-running** editor domain — no domain
reload — so **play-mode state and object references survive** and the change is live in under a second.
It's the antidote to the bridge's slowest loop: edit a tuning value in a `MonoBehaviour`, normally pay
a recompile + domain reload (seconds, exits/re-enters play, drops all runtime state), just to see one
number change. With Live Patch you edit the method, `live_patch_apply`, and keep playing.

This is a power tool with sharp, honest limits. Read the boundaries before you lean on it.

## When Live Patch applies vs. falls back

Live Patch classifies your edit by diffing the new source against what's on disk:

- **Body-only change → hot-applied.** You changed the *inside* of one or more methods / property
  accessors. These are compiled in-memory and detoured onto the live methods. State preserved.
- **Structural change → falls back to a normal recompile (a real domain reload).** Anything that
  changes the type's shape: adding/removing/renaming a field or member, changing a method signature
  (params/return type/`static`), adding a type, changing a base class/interface. There is **no way**
  to hot-apply these — the live objects' memory layout would no longer match. Live Patch detects this,
  does **not** patch, and tells you it needs a reload (it writes the file so Unity recompiles it).

It is **conservative by design**: when in doubt it calls a change structural. A false "hot-applied"
would silently corrupt behavior; a false "structural" just costs you one ordinary reload.

## Edits it will SKIP (and report) even when body-only

Some method shapes can't be soundly detoured, so Live Patch skips them with a reason rather than
mispatch — you'll see them in the `skipped` list and the method keeps its old behavior until a real
reload:

- generic methods or methods on generic/`struct`/`record` types
- `async` methods, iterators (`yield`)
- methods that use `base.` 
- constructors and operators (v1 supports plain methods + property get/set/init)

If your tweak lands in one of these, just edit-and-recompile normally.

## The loop

Pair this with `claude-bridge-build-feature` (the screenshot-driven discipline) — Live Patch only
changes *how fast* the inner loop turns; you still **verify** every change.

1. **Confirm capability.**
   ```
   mcp__claude-bridge-unity__live_patch_status
   ```
   Reports `available` (is the package installed?), `supported` (editor is Mono → yes), `isPlaying`,
   whether MonoMod + Roslyn resolved, and the active-patch count. If `available:false`, the package
   isn't installed — fall back to the normal `editor_recompile` loop, or install it (Package Manager →
   Add package from disk → `unity/com.lousputthole.livepatch/package.json`).

2. **Enter play mode and build up the state you want to iterate against** (`sim_play`, then poll
   `sim_status`). The whole point is to keep this state across edits.

3. **Edit the method body** with the normal file tools (`Edit`/`read_file`), then apply it:
   ```
   mcp__claude-bridge-unity__live_patch_apply  assetPath="Assets/Scripts/Player.cs" newSource="<full new file content>"
   ```
   - Pass the **full new file content** as `newSource` (it's diffed against the file's current on-disk
     content). The tool returns per-method `patched` / `skipped` / `failed` lists and an overall
     `verdict` (`hot-reloaded` | `structural` | `skipped` | `none`).
   - `writeToDisk` defaults true (the file is persisted so the change isn't lost; Unity reconciles it
     with a normal recompile when you stop play). Pass `writeToDisk=false` to patch the live domain
     **without** touching the file (throwaway experiments).

4. **Observe immediately** — this is why you're here. `screenshot_game` and read the PNG; or read live
   values with `get_component_fields`; or fire logic with `invoke_method`. No reload happened, so the
   same objects are running the new code.

5. **Iterate.** Re-edit, re-`live_patch_apply`. Re-applying the same method just replaces the detour.

6. **Revert when needed.** `live_patch_revert` with an empty signature drops **all** patches back to the
   original compiled IL (state preserved); pass an exact signature (from `live_patch_list`) to revert
   one. A real domain reload (recompile, exit play) also clears every patch automatically.

## Honest expectations

- **Editor only.** Live Patch works in the Unity Editor (always Mono). It does **not** work in IL2CPP
  builds (AOT, no JIT) — that's a permanent limit, not a v1 gap.
- **It is not a substitute for recompiling.** When you're done iterating, the patched file is on disk;
  let Unity do a normal recompile (stop play, or `editor_recompile`) so the real assembly matches what
  you'll ship. Live Patch is a fast *inner* loop, not the final state.
- **Verify like everything else.** A patch that `failed` left the old behavior in place. Read the
  result envelope; if a method is in `failed`/`skipped`, it did **not** change — don't assume it did.
- **`execute_csharp` is still the tool for one-off editor scripting** (run an importer, a batch edit).
  Live Patch is specifically for *changing the behavior of code that's already running*.

## When it falls back, say so

If `verdict` comes back `structural`, tell the user plainly: the edit changed the type's shape, so it
needs a normal recompile (one domain reload) — and then run the standard `editor_recompile` →
`wait_ready` → `get_compile_errors` loop. Don't keep retrying `live_patch_apply` on a structural edit;
it will never hot-apply.
