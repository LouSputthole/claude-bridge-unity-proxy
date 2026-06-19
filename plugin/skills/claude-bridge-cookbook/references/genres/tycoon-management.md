# Tycoon / Management / Idle — Genre Recipe

A numbers-go-up economy: a cheap repeated action → currency → geometric-cost upgrades → (optionally) prestige. Built from the bridge's economy, save, interactable, and day-night scaffolds.

## The loop

Do a cheap action (collect / produce) → earn currency → spend on **geometric-cost upgrades** that make the action faster/more profitable → unlock new content → (optional) **prestige** reset for a permanent multiplier. Everything else (placeable producers, idle accrual, a daily cycle) bolts onto that spine.

## The system stack to compose

1. **Currency / wallet** — `references/systems/economy-currency.md` (`create_economy_wallet`). The one place money is minted/spent.
2. **Upgrades + balance tables** — `references/systems/progression-upgrades.md` (`create_stat_modifier_system` + a ScriptableObject cost table). Geometric costs, applied to live producers.
3. **The action** — either an `create_interactable` you click, or placeable producers (`references/systems/building-placement.md`) that tick income.
4. **Save** — `references/systems/save-load.md` (`create_save_system`). Clamp every field on load.
5. **Day/night or daily cycle** (optional) — `references/systems/day-night.md` (`create_day_night_clock`) for daily rent/income beats.
6. **HUD** — `references/systems/ui-hud.md` (money label bound to `OnBalanceChanged`, an upgrade screen).

## Bridge build order

Build the *loop* before the meta.

**1. Block out the scene.** A ground plane (`game_object_create_primitive type="Plane"`) + a couple of primitive "producers." `screenshot_scene`, read it, confirm framing.

**2. Phase A — generate the spine, then compile:**
- `create_economy_wallet name="Wallet" currencyName="Gold" startingBalance=0`
- `create_stat_modifier_system name="UpgradeStats" statsCsv="IncomeRate,ClickPower"`
- `create_interactable name="Producer" prompt="Click to collect" range=4` (or skip if producers are placed buildings)
- `create_save_system name="SaveSystem"`
- `create_day_night_clock name="DayClock" dayLengthSeconds=120` (optional)
- `editor_recompile` → `wait_ready` → `get_compile_errors`. **Fix red before Phase B.**

**3. Phase B — place + wire:**
- `component_add target="GameManager" type="Wallet"`, `… type="UpgradeStats"`, `… type="SaveSystem"`.
- Producers: `component_add` the interactable/income script; clicking or ticking calls `Wallet.Add(amount)`, where `amount` reads the `IncomeRate`/`ClickPower` stat.
- Shop: a buy action does `Wallet.TrySpend(cost)` (cost from the table) → adds a modifier to the stat → producers immediately earn more.
- HUD: `create_canvas` + a money `create_ui_text` bound to `Wallet.OnBalanceChanged`; an upgrade `create_ui_button` calling the buy.
- Save: the POCO reads `Wallet` balance + upgrade levels; load clamps and re-applies.

## How to verify each step

- **Wallet:** `sim_play` → `invoke_method target="GameManager" componentType="Wallet" methodName="Add" argsJson="[100]"` → `get_runtime_property … propertyPath="Balance"` = 100. `TrySpend(150)` → false; `TrySpend(60)` → true, balance 40.
- **Earning:** with a producer wired, run a few seconds (or `sim_set_speed speed=8`) and watch the balance climb via repeated `get_runtime_property`.
- **Upgrade:** `invoke_method` the buy, confirm balance dropped AND the income/click stat rose (`get_runtime_property` both). That's the core loop proven.
- **Save round-trip:** bump money, `invoke_method … SaveSystem.Save`, `sim_stop`, `sim_play`, `… Load`, read balance back — it matches. (See `references/systems/save-load.md`.)
- **Daily cycle:** `sim_set_speed speed=20`, `screenshot_game` at intervals, read the PNGs for the lighting swing; confirm a daily income/rent event fired via `get_runtime_property`.
- **HUD:** `screenshot_game`, read the money label; change money and confirm it updates.

## Gotchas (genre-specific; the cross-cutting laws are in SKILL.md)

- **`int` wallet caps ~2.1B** — for deep idle numbers, hand-write a `long`/`double` wallet and `NaN`-guard every delta (see `references/systems/economy-currency.md`).
- **Adding an upgrade-table row invalidates higher saved levels** — clamp on load.
- **Re-apply upgrade stats on load** or loaded producers earn at base rate.
- **Spend through `TrySpend` only**, never `SetBalance` minus cost.
- **The bridge can't feel pacing** — whether the upgrade curve is satisfying is a human playtest call.

## Cross-links

Systems: economy-currency, progression-upgrades, save-load, building-placement, day-night, ui-hud. For placed-producer tycoons lean on `references/systems/building-placement.md`. For the whole workflow, `claude-bridge-build-feature`.
