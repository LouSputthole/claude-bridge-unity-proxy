# Economy / Currency / Wallet

Hold a currency on one authority component, validate every spend, fire a change event the HUD binds to. Built on the real `create_economy_wallet` scaffold.

## What it IS / when you need it

A single `Wallet` Component holding an integer balance with `Add` / `TrySpend` / `SetBalance` / `CanAfford` and an `OnBalanceChanged` event. It is the *one place* money is minted or spent — every shop, reward, and cost routes through it so you never have two code paths disagreeing about the balance.

## Bridge build order

**Phase A — generate + compile:**
1. `create_economy_wallet name="Wallet" startingBalance=0 currencyName="Gold"` — generates the wallet MonoBehaviour (balance + `Add`/`TrySpend`/`SetBalance`/`CanAfford` + `OnBalanceChanged`).
2. `editor_recompile` → `wait_ready` → `get_compile_errors`.

**Phase B — place + wire:**
3. `component_add target="GameManager" type="Wallet"` (or the player object, depending on whether currency is global or per-player).
4. `get_component_fields target="GameManager" componentType="Wallet"` — confirm `currencyName` and starting balance.
5. Route every earn/spend through it: rewards call `Add`, purchases call `TrySpend(cost)` and branch on the bool. **Never mutate the balance field directly from two places.**
6. HUD: subscribe a label to `OnBalanceChanged` (see `references/systems/ui-hud.md`).

## Verify

- **Edit-scene:** `get_component_fields` shows the starting balance + currency name.
- **Runtime:** `sim_play` → poll `sim_status`. `invoke_method target="GameManager" componentType="Wallet" methodName="Add" argsJson="[100]"` then `get_runtime_property … propertyPath="Balance"` → 100. Test the spend gate: `invoke_method … methodName="TrySpend" argsJson="[150]"` should return `false` (can't afford) and leave the balance at 100; `argsJson="[60]"` returns `true` and drops it to 40. This proves the validation, not just the call.

## Variations

- **Multiple currencies** (soft + premium): one `Wallet` instance per currency, distinguished by `currencyName`, or a small manager holding several wallets. Don't cram two balances into one component.
- **Big numbers** (idle games): the scaffold is `int`. If you need to exceed ~2.1 billion, hand-write a `long`/`double` variant and clamp/`NaN`-guard every delta — a single bad multiply poisoning a saved balance is the classic idle-game corruption bug.
- **Geometric upgrade costs** belong in a balance table read by the shop, not in the wallet — see `references/systems/progression-upgrades.md`.
- **Persistence:** the wallet's balance is a field your save POCO reads/writes — see `references/systems/save-load.md`. Clamp on load.

## Gotchas

- **Always spend through `TrySpend`, never `SetBalance` minus a cost** — `TrySpend` is the affordability gate; bypassing it lets the balance go negative.
- **Two-phase:** `create_economy_wallet` → recompile → THEN `component_add`.
- **`OnBalanceChanged` must fire on every mutation** (the scaffold does this) or the HUD silently drifts from the real value — verify with `get_runtime_property` after a change, not by watching the UI.
- **Don't trust an optimistic UI number** for an irreversible action — read the authoritative balance back.

## Cross-links

`references/systems/progression-upgrades.md` (cost curves + buying), `references/systems/ui-hud.md` (the money label), `references/systems/save-load.md` (persist it). Genres: `references/genres/tycoon-management.md`, `references/genres/rpg.md`, `references/genres/tower-defense.md`.
