# Save / Load / Persistence

JSON save-and-load to disk through the bridge, built on the real `create_save_system` scaffold. The discipline that matters: **version + clamp on load**, because saves outlive your balance changes.

## What it IS / when you need it

A Component that serializes a plain `[System.Serializable]` save POCO to `Application.persistentDataPath` and reads it back. Reach for it the moment you have any progress worth keeping across sessions (currency, levels, unlocks, placed buildings, quest state).

## Bridge build order

**Phase A — generate + compile:**
1. `create_save_system name="SaveSystem"` — generates a SaveSystem MonoBehaviour doing JSON save/load to `persistentDataPath` (uses `JsonUtility` + `System.IO.File`, the right runtime-safe path — NOT `AssetDatabase`, which is editor-only).
2. `editor_recompile` → `wait_ready` → `get_compile_errors`.

**Phase B — define your payload + wire:**
3. Decide what to persist. The scaffold gives you the read/write plumbing; you fill the save POCO with the fields your game owns (money, level, slot list, etc.). Extend the generated file with `edit_script` (literal find/replace) or add a sibling `[System.Serializable]` data class via `create_script`. **Add a `int Version` field on day one.**
4. `component_add target="GameManager" type="SaveSystem"` after the recompile.
5. Wire each system to push its state into the POCO on save and read it back on load — `create_economy_wallet`'s balance, `create_inventory`'s slots, etc. **Clamp every field on load** (`Mathf.Clamp`, re-resolve ids against the live catalog, drop unknown entries).

## Verify

- **Round-trip in play mode:** `sim_play` → poll `sim_status`. Set some live state (`set_runtime_property` / `invoke_method` to bump money or add an item), then `invoke_method target="GameManager" componentType="SaveSystem" methodName="Save"`. `sim_stop`. `sim_play` again and `invoke_method … methodName="Load"`, then read the values back with `get_runtime_property` — they should match what you saved.
- **Confirm the file exists** with `editor_read_log` if the scaffold logs the path, or just trust the read-back round-trip — that's the real proof.
- Remember **runtime-only changes are discarded on `sim_stop`** — that's exactly why the save file (which lives on disk, not in the scene) is the thing that proves persistence.

## Variations

- **Versioned migration:** branch on `save.Version` when reading; bump it when the schema changes; clamp/migrate older payloads instead of trusting them. The simplest valid policy is "if `Version != Current`, reset to defaults"; the nicer one migrates field-by-field.
- **Multiple slots:** key the filename by slot index (`save_0.json`, `save_1.json`); a tiny sidecar meta file (`{timestamp, money}`) lets a load menu list slots without parsing the full save.
- **Autosave:** a dirty flag + a coroutine/`Update` timer that calls `Save` when set — don't save every frame.
- **ScriptableObject config vs save:** balance tables that ship with the game are `create_scriptable_object` assets (authored, read-only at runtime); *player* state is JSON on `persistentDataPath`. Don't conflate them.

## Gotchas

- **Never use `AssetDatabase` for player saves** — it's editor-only and absent in a build. The scaffold already uses `persistentDataPath`; keep any editor-only calls behind `#if UNITY_EDITOR`.
- **`JsonUtility` won't serialize a `Dictionary`, an interface, or a raw `null`-able polymorphic field.** Use `[System.Serializable]` lists of concrete structs/classes. The scaffold is built around this; honor it when you extend the POCO.
- **Clamp on load or a stale save crashes/cheats you** — a level index past a shrunk table, a negative balance, a removed item id. Sanitize before applying.
- **Two-phase:** `create_save_system` → recompile → THEN `component_add`. Editing the POCO is another `.cs` change — recompile again before relying on the new fields.

## Cross-links

Every stateful system feeds this: `references/systems/economy-currency.md`, `references/systems/inventory.md`, `references/systems/progression-upgrades.md`, `references/systems/objectives-quests.md`, `references/systems/building-placement.md`. For the genre wiring, see `references/genres/tycoon-management.md` and `references/genres/rpg.md`.
