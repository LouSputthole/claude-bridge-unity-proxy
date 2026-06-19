# Progression / Upgrades / Stats

Data-driven balance tables, geometric-cost upgrades, and a modifiable stat system. Built on the real `create_stat_modifier_system` scaffold plus ScriptableObject balance tables.

## What it IS / when you need it

Two layers that work together:
- **Stats** — a `create_stat_modifier_system` Component holding base values with additive + multiplicative modifiers keyed by source (so gear/buffs/upgrades can each contribute and be removed cleanly).
- **Upgrades** — a balance table (costs + effects per level) the shop reads; buying spends currency and applies a modifier or bumps a level.

Reach for it whenever numbers grow: damage/speed/health upgrades, tycoon producers, RPG character sheets.

## Bridge build order

**Phase A — generate + compile:**
1. `create_stat_modifier_system name="PlayerStats" statsCsv="Health,Speed,Damage"` — generates a `<Name>Stat` enum (e.g. `PlayerStatsStat.Health`), base values, and additive/multiplicative modifiers keyed by source, with an `OnStatChanged` event.
2. (Optional) `create_scriptable_object` for a balance table asset, or a `[System.Serializable]` config array in a `create_script` file — costs/effects per level.
3. `editor_recompile` → `wait_ready` → `get_compile_errors`.

**Phase B — place + wire:**
4. `component_add target="Player" type="PlayerStats"`.
5. Consumers read the resolved stat: the health system's max = `PlayerStats.Get(Health)`, the controller's speed = `PlayerStats.Get(Speed)`, etc. (wire via small glue or by reading the stat in those components).
6. The shop/upgrade flow: `Wallet.TrySpend(cost)` → on success add a modifier (`AddModifier(stat, source, value)`) or increment a level → the stat's `OnStatChanged` propagates to consumers.

## Verify

- **Runtime:** `sim_play` → poll `sim_status`. Read a base stat: `get_runtime_property target="Player" componentType="PlayerStats" propertyPath="<Speed value>"`. Apply a modifier: `invoke_method target="Player" componentType="PlayerStats" methodName="AddModifier" argsJson="[…]"` (confirm the real signature with `get_method_signature` after compile) and re-read — the resolved value should change. Remove the modifier by source and confirm it reverts. That round-trip is the proof the source-keyed removal works.
- **Cost curve:** `invoke_method` the shop's "get cost for level N" and confirm the geometric growth, then buy and confirm currency dropped + stat rose.

## Variations

- **Geometric cost:** `cost(level) = baseCost * mult^level`. Keep it in the table, read by the shop — not hard-coded in the wallet.
- **Prestige / rebirth:** bank a permanent multiplier from current totals, reset the run-level stats/levels, bump a prestige counter, save. The multiplier feeds back as a modifier source.
- **Maxed upgrades:** return a sentinel (e.g. `-1`) for "no next level" so the UI can grey out the button.
- **Stacking sources:** the scaffold keys modifiers by source — gear, a temporary potion, and a permanent upgrade can all modify `Damage` and each be removed independently. Lean on that instead of one mutable number.

## Gotchas

- **Adding a table row invalidates saves above the old max** — your save loader MUST clamp the persisted level to the new `MaxLevel` on load (see `references/systems/save-load.md`).
- **Re-apply derived stats on load** — if buying mutates live components, loading must replay every owned upgrade so the live values match the saved levels.
- **Two-phase:** `create_stat_modifier_system` → recompile → THEN `component_add`. The generated enum (`<Name>Stat`) only exists after the compile — don't reference it before.
- **Don't store the *resolved* stat in the save** — store the levels/owned-upgrades and re-derive; the resolved value is a function of them and your current balance table.

## Cross-links

`references/systems/economy-currency.md` (paying for upgrades), `references/systems/health-combat.md` (stats feed damage/health), `references/systems/save-load.md` (persist levels, clamp on load), `references/systems/ui-hud.md` (the upgrade screen). Genres: `references/genres/tycoon-management.md`, `references/genres/rpg.md`.
