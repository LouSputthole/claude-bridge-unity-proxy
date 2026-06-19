# Inventory (slots, hotbar, pickup & drop)

The "player holds items" system in Unity through the bridge: a Component that owns the items, raises a change event the HUD binds to, plus world pickups that feed it. Built on the real `create_inventory` + `create_pickup` scaffolds.

## What it IS / when you need it

A Component on the player GameObject that owns a fixed set of slots, exposes mutators (`Add`/`Remove`/`Has`/`Count`/`IsFull`), and fires `OnChanged` so the UI updates. Reach for it when items have identity (a tool, a gun + ammo, stacks of ore). If you only need a few global counters, a couple of `int` fields on a manager is enough — don't build this.

## Bridge build order

**Phase A — generate + compile (never skip the compile gate):**
1. `create_inventory name="PlayerInventory" slots=20` — generates a slot-based MonoBehaviour: `Add`/`Remove`/`Has`/`Count`/`IsFull` + `OnChanged`. (Items are tracked as string ids/keys per slot — the scaffold's storage is slot-based, not a serialized `Dictionary`, which is exactly what survives serialization.)
2. `create_pickup name="ItemPickup" kind="ammo"` — generates a trigger collectible that raises `OnPickedUp` then destroys/disables itself.
3. `editor_recompile` → `wait_ready` → `get_compile_errors`. Fix any red against YOUR files before continuing.

**Phase B — place + wire:**
4. Player object: `component_add target="Player" type="PlayerInventory"`.
5. Confirm members with `get_component_info target="Player"` and `get_component_fields target="Player" componentType="PlayerInventory"` — read back the slot count.
6. Make a pickup: `game_object_create_primitive type="Sphere" name="AmmoPickup"`, then `component_add type="ItemPickup"`, then a trigger collider — `physics_add_collider target="AmmoPickup" type="Sphere"` and set `isTrigger` via `component_set target="AmmoPickup" componentType="SphereCollider" propertiesJson="{\"isTrigger\":true}"`.
7. Wire the pickup's grant into the inventory. The clean coupling is a tiny glue script (`create_script`) on the pickup whose `OnPickedUp` handler calls `inventory.Add(...)`. Compile-verify it like any generated code.

## Verify

- **Edit-scene:** `get_component_fields` to confirm the inventory's slot count and the pickup's kind took. `frame_object target="AmmoPickup"` + `screenshot_scene` to see it placed on the ground (use `snap_to_ground target="AmmoPickup"` first so it isn't floating).
- **Runtime:** `sim_play`, poll `sim_status` until playing. Walk/teleport the player onto the pickup (`teleport target="Player" position="<pickup pos>"`), then read the result with `get_runtime_property target="Player" componentType="PlayerInventory" propertyPath="Count"` — the count should rise and the pickup GameObject should be gone (`find_in_play query="AmmoPickup"` returns nothing). You can also force a grant directly: `invoke_method target="Player" componentType="PlayerInventory" methodName="Add" argsJson="[\"ammo\",1]"` and re-read `Count`.

## Variations

- **Capacity policy** is `IsFull` on the scaffold (slot count). For weight/encumbrance, hand-write a `float` running total against a `MaxWeight` and fire `OnChanged` — don't try to bolt it onto the slot scaffold.
- **Hotbar selection** = an `int ActiveSlot` + reading number-key input in `Update` (guard input behind whatever "is this the local player" check your project uses). Add it as a small companion script that reads the inventory.
- **Per-instance condition** (wear/uses/rolled suffix): wrap the item id in a `[System.Serializable]` struct carrying the extra fields; keep it a *list*, never a `Dictionary`, so it serializes.
- **Drop** = the inverse of pickup: instantiate the item's world prefab at the player (`prefab_instantiate`), then `Remove` from the inventory.

## Gotchas

- **The scaffold compiles standalone but isn't placed for you.** Two-phase: `create_inventory` → recompile → THEN `component_add`. Adding the component before the domain reload fails to resolve the type.
- **A `Dictionary` won't serialize.** The scaffold avoids this; if you extend it, store a `[System.Serializable]` list of structs, not a dict.
- **Pickup needs a trigger collider AND the other body needs a collider** for `OnTriggerEnter` to fire. Confirm both with `get_component_info`. A pickup with no collider silently never triggers.
- **String item keys are fragile** across asset renames — persist a stable id, re-resolve display data from a static table on load.
- **Don't gate irreversible economy on an optimistic read** — confirm the authoritative count after a grant.

## Cross-links

`references/systems/economy-currency.md` (currency is a one-value inventory), `references/systems/save-load.md` (snapshot the inventory), `references/systems/ui-hud.md` (bind a slot HUD to `OnChanged`). For exact member signatures, `describe_type` your generated `PlayerInventory` after it compiles.
