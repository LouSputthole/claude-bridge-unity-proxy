# Audio (SFX, music, ambience)

Wire AudioSources, a listener, and play/stop control through the real Audio family tools.

## What it IS / when you need it

Sound = **a listener** (the scene's "ears," usually on the main camera), **AudioSources** (2D for UI/music, 3D/spatial for world sounds), and **triggers** (play on event). Reach for it for footsteps, weapon fire, pickups, ambient loops, music, UI clicks.

## Bridge build order

This is scene authoring + light wiring.

1. **Listener first** — without it you hear nothing: `add_audio_listener target="Main Camera"`. (If the camera already has one, the tool is a no-op; confirm with `get_component_info target="Main Camera"`.)
2. **Music / ambience** (2D, looping): `create_audio_source_object name="Music" clipPath="<asset>"` (a standalone audio object), or `add_audio_source target="GameManager" clipPath="<asset>" loop=true playOnAwake=true volume=0.5`. Leave `spatial=false` for 2D.
3. **World SFX** (3D): `add_audio_source target="<thing>" clipPath="<asset>" spatial=true` so it attenuates with distance, then `play_sound target="<thing>"` on the event (or wire it from code).
4. **Find clips:** `find_assets query="<name>" type="AudioClip"` to get the project path for `clipPath`. `set_audio_source target="…" propsJson="{…}"` adjusts fields (pitch, volume, spatialBlend) after the fact.

## Verify

- **The bridge has no ears** — you can't "hear" the result. Verify *structurally*:
  - `get_component_info target="<obj>"` shows the AudioSource is present and `get_component_fields … componentType="AudioSource"` shows the clip is assigned, loop/volume/spatial are right, and (for music) `playOnAwake` is set.
  - Confirm the listener exists exactly once: `get_component_info target="Main Camera"` (multiple listeners warn in Unity; `scene_validate` flags scene issues).
  - **Runtime:** `sim_play` → `invoke_method`/`play_sound` to trigger a one-shot, and read `get_runtime_property target="<obj>" componentType="AudioSource" propertyPath="isPlaying"` → true. That's the honest "it fired" check.
- State plainly to the user that you've wired and confirmed the sources but **can't judge mix/timing/whether it sounds right** — that's their ear.

## Variations

- **One-shots vs loops:** footsteps/hits are one-shots (`play_sound` per event, `loop=false`); music/ambience loop. Don't loop a one-shot.
- **2D vs 3D:** UI and music are 2D (`spatial=false`); world sounds are 3D (`spatial=true`) so position matters — a 3D source on a moving object follows it.
- **Pitch variation** to avoid machine-gun repetition: nudge `pitch` per play via `set_audio_source`.
- **Generated audio:** if you need actual sound files, the ElevenLabs `sound-effects` / `music` skills can produce clips to drop into the project, then `find_assets` + assign.

## Gotchas

- **No AudioListener = silence**, and **two listeners = a warning/garbled audio.** Exactly one, usually on the main camera. `scene_validate` / `get_component_info` catch both.
- **`clipPath` must resolve** — use `find_assets type="AudioClip"` to get the real path; a wrong path leaves the source clip-less (visible in `get_component_fields`).
- **`playOnAwake` only fires in play mode** — an edit-scene check won't "play" it; confirm with the runtime `isPlaying` read.
- **Spatial sound needs the source positioned in the world** — a 3D source at the origin sounds wrong; parent it to its emitter.

## Cross-links

Most systems trigger sound on their events: `references/systems/health-combat.md` (hit/death), `references/systems/inventory.md` (pickup), `references/systems/ui-hud.md` (clicks), `references/systems/day-night.md` (ambience per phase). The Audio tools are the Audio family in `docs/TOOLS.md`.
