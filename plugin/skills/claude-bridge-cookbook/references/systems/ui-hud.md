# UI / HUD (uGUI)

Build a screen-space HUD with the real uGUI tools — Canvas, text, buttons, panels, sliders — and bind it to your game's change events.

## What it IS / when you need it

The on-screen layer: health bar, money counter, ammo, objective list, menus, dialogue box. This bridge's UI family builds **uGUI** (Canvas + RectTransform), and auto-ensures an EventSystem so buttons are clickable.

## Bridge build order

This is a **scene-authoring** task, not codegen — most of it needs no recompile (you're creating GameObjects + components). Wiring the bindings is the only code part.

1. `create_canvas name="HUD" renderMode="ScreenSpaceOverlay"` — creates the Canvas (CanvasScaler + GraphicRaycaster) and ensures a scene EventSystem. (`ensure_event_system` if you ever need it standalone.)
2. Build elements under it:
   - `create_ui_text parent="HUD" text="Gold: 0" name="MoneyLabel"`
   - `create_ui_panel parent="HUD" color="0,0,0,0.5" name="BottomBar"` (a stretched panel)
   - `create_ui_button parent="HUD" label="Upgrade" name="UpgradeBtn"`
   - `create_ui_slider parent="HUD" name="HealthBar"` (range 0..1 — perfect for `HealthFraction`)
   - `create_ui_image parent="HUD" color="1,1,1,1" name="Crosshair"`
3. Position each with `set_rect_transform target="MoneyLabel" anchorMin="0,1" anchorMax="0,1" pivot="0,1" anchoredPosition="20,-20"` (each `"x,y"` param applies only if non-empty). Color text/images with `set_ui_color`.
4. **Bind to game events** (the code part): a small HUD script (`create_script`) that subscribes to `Wallet.OnBalanceChanged` → `set_ui_text` on MoneyLabel, `Health.OnDamaged`/`OnHealed` → set the HealthBar slider value, etc. Compile-verify it (`editor_recompile` → `wait_ready` → `get_compile_errors`), then `component_add` it.

## Verify

- **See it.** `screenshot_game` (Screen-Space-Overlay renders to the game camera) and **read the PNG** — uGUI anchoring/pivots are the single easiest thing to get subtly wrong (label off-screen, panel covering everything). Iterate `set_rect_transform` against what you see, not what you assume.
- **The binding:** `sim_play` → poll `sim_status`. Change the source (`invoke_method` on the Wallet to `Add` money) and confirm the label updated — read it with `get_runtime_components target="MoneyLabel"` (or just `screenshot_game` and read the text). If the number doesn't move, the event isn't firing or isn't bound.
- **Buttons:** confirm the EventSystem exists (the Canvas tool ensures it). A dead button usually means no EventSystem or the button's `onClick` isn't wired.

## Variations

- **World-space UI** (health bars over enemies, nameplates): `create_canvas renderMode="WorldSpace"`, scale it down, parent it to the unit. Screenshots show whether it's billboarded right.
- **Menus / pause:** a full-rect `create_ui_panel` with buttons; toggle its GameObject active for show/hide (`game_object_set_active`).
- **Bars from sliders:** `create_ui_slider` is the cheap health/progress bar — drive its value 0..1 from `HealthFraction` or objective progress.
- **Dynamic lists** (inventory grid, objective list): instantiate a row prefab per item under a panel; rebuild on the source's `OnChanged`.

## Gotchas

- **You don't see Console/Game output** — verify a binding by changing the source and reading the label back (`screenshot_game` or `get_runtime_components`), not by assuming.
- **RectTransform anchors/pivots are unintuitive** — `set_rect_transform`'s `"x,y"` strings only apply when non-empty, so you can set anchors without disturbing size. Screenshot after each change.
- **No EventSystem = dead buttons.** `create_canvas` ensures one; if you built UI another way, `ensure_event_system`.
- **Overlay vs Camera vs World** render differently — `screenshot_game` shows Overlay/Camera; World-space UI you frame like any object. Pick the render mode deliberately.
- **Bind on the event, don't poll in `Update`** for values like money — it churns and can still drift; the event is the source of truth.

## Cross-links

Every system has a HUD face: `references/systems/economy-currency.md` (money label), `references/systems/health-combat.md` (health bar), `references/systems/objectives-quests.md` (objective list + `create_objective_marker`), `references/systems/inventory.md` (slot grid), `references/systems/dialogue.md` (dialogue box). The UI tools are the UI family in `docs/TOOLS.md`.
