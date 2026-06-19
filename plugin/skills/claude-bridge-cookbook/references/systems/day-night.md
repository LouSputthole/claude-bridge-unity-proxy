# Day / Night Cycle

A normalized time-of-day that rotates a sun and fires dawn/dusk events. Built on the real `create_day_night_clock` scaffold + the atmosphere/lighting tools.

## What it IS / when you need it

A `DayNightClock` Component holding a normalized time-of-day that rotates a directional (sun) Light, exposes `Hour`, and fires `OnHourChanged`/`OnDawn`/`OnDusk`. Reach for it for survival games (threats at night), farming/tycoon (daily cycles), or just mood.

## Bridge build order

**Phase A — generate + compile:**
1. `create_day_night_clock name="DayNightClock" dayLengthSeconds=120` — generates the clock with the events above (real seconds per in-game day).
2. `editor_recompile` → `wait_ready` → `get_compile_errors`.

**Phase B — place + wire:**
3. Ensure a sun: `add_light type="Directional" name="Sun" intensity=1` if the scene has none.
4. `component_add target="DayNightClock" type="DayNightClock"` (an empty manager object is fine) and point its sun field at the directional Light (`component_set` with the Light reference, or whatever the scaffold's field is — read it with `get_component_fields`).
5. Hook the events: `OnDusk` could enable a `create_spawner` of night creatures, dim ambient, raise fog; `OnDawn` reverses it.

## Verify

- **The rotation, visually:** `sim_play` → poll `sim_status`. `sim_set_speed speed=20` so a full day passes in seconds. Take `screenshot_game` at intervals and read the PNGs — the lighting should swing from day to dusk to night. (Aim the camera at the scene first, or use `screenshot_scene`.)
- **The clock value:** `get_runtime_property target="DayNightClock" componentType="DayNightClock" propertyPath="Hour"` over a few reads should advance. Force a time to test an event boundary: `set_runtime_property … property="<time field>" value="<dusk>"` and confirm the `OnDusk` effect fired (spawner enabled, fog raised).
- You can also drive mood directly for a screenshot: `apply_atmosphere mood="foggy-dawn"` / `apply_atmosphere mood="horror-night"` composes ambient + fog + sun + skybox in one call — handy to preview the *look* a clock phase should produce before wiring it.

## Variations

- **Atmosphere per phase:** call `apply_atmosphere` (or `set_ambient` + `set_fog` + `set_skybox_tint`) from the dawn/dusk handlers for a full mood swing, not just a sun rotation.
- **Gameplay gates:** night = enemies spawn / shops close / a curfew. Wire those off `OnDusk`/`OnDawn`.
- **Faster/slower days:** `dayLengthSeconds` at generation, or expose it as a runtime field.
- **Calendar:** count days in a manager for "survive N nights" or farming seasons.

## Gotchas

- **The clock needs a directional Light to rotate** — if nothing visibly changes, the sun field is unset. Confirm with `get_component_fields`; `add_light type="Directional"` if missing.
- **Built-in RP fog/ambient** is what `set_fog`/`set_ambient`/`apply_atmosphere` drive — this bridge targets Built-in RP, so those tools apply to the right pipeline. Don't expect URP/HDRP volume behavior.
- **Two-phase:** `create_day_night_clock` → recompile → THEN `component_add`.
- **Time scale affects everything** — `sim_set_speed` to fast-forward the day also speeds physics/AI; reset with `sim_reset` or `sim_set_speed speed=1`.
- **Events fire on threshold crossings** — to test, set the time *just before* dawn/dusk and step, rather than expecting a mid-day read to fire them.

## Cross-links

`references/systems/spawning-waves.md` (night spawns), `references/genres/survival.md` (survive the night), `references/genres/horror.md` (`apply_atmosphere mood="horror-night"`), `references/genres/tycoon-management.md` (daily cycles). Atmosphere tools live in the Visuals family of `docs/TOOLS.md`.
