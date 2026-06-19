# Objectives / Quests

Named goals with progress, completion events, and on-screen markers. Built on the real `create_objective_system` + `create_objective_marker` scaffolds, often fed by `create_trigger_zone`.

## What it IS / when you need it

An `Objectives` Component holding named objectives with progress, firing `OnObjectiveCompleted`/`OnAllComplete`. Pair it with `create_objective_marker` for an on-screen waypoint and `create_trigger_zone` for "reach this place" goals. Reach for it for quests, tutorial steps, level win-conditions, checklists.

## Bridge build order

**Phase A — generate + compile:**
1. `create_objective_system name="Objectives"` — generates the objectives MonoBehaviour (named objectives + progress + `OnObjectiveCompleted`/`OnAllComplete`).
2. (Optional) `create_objective_marker name="QuestMarker"` — projects a world position to screen space for a UI `RectTransform`, clamps to screen edges, shows distance + label + color (the floating waypoint).
3. (Optional) `create_trigger_zone name="GoalZone"` — a trigger BoxCollider raising `OnEnter`/`OnExit` with a tag filter + once-only; the canonical "reach here" objective.
4. `editor_recompile` → `wait_ready` → `get_compile_errors`.

**Phase B — place + wire:**
5. `component_add target="GameManager" type="Objectives"`; register the objectives (via inspector fields / an init call — read the real API with `get_component_fields` / `describe_type` after compile).
6. For a location goal: place the zone (`game_object_create` + the trigger script, or it self-adds its collider), set its tag filter to `Player` (the player must carry that tag — `add_tag` + `game_object_set_tag` first), and have `OnEnter` advance the matching objective.
7. For a waypoint: `create_canvas`, add the marker under it, point it at the goal's transform.
8. `OnAllComplete` → win state (`create_game_manager` state change, see `references/genres/puzzle.md`).

## Verify

- **Edit-scene:** `get_component_fields` shows the registered objectives. `frame_object target="GoalZone"` + `screenshot_scene` to see the trigger volume placed.
- **Runtime:** `sim_play` → poll `sim_status`. Drive an objective directly: `invoke_method target="GameManager" componentType="Objectives" methodName="<complete/advance>" argsJson="[…]"` (confirm the signature with `get_method_signature`), then `get_runtime_property` the progress. For the trigger goal, `teleport target="Player" position="<inside GoalZone>"` and confirm the objective advanced (read it back) — this proves the tag filter + `OnEnter` wiring without needing to walk there. `screenshot_game` to see the marker on screen.

## Variations

- **Counted objectives** ("collect 5"): progress increments on each pickup (`create_pickup`'s event), completes at the target.
- **Sequential quests:** complete one to unlock the next; the system fires `OnObjectiveCompleted`, a quest manager activates the follow-up.
- **Dialogue-given quests:** a `DialogueRunner` choice registers/advances an objective — see `references/systems/dialogue.md`.
- **Markers for multiple goals:** one `create_objective_marker` per active goal, each pointed at its transform; they clamp to screen edges when off-screen.

## Gotchas

- **The trigger goal needs the player tagged + both colliders present.** `add_tag "Player"` → `game_object_set_tag target="Player" tag="Player"`, and the zone's filter must match. A goal that never fires is usually a missing tag or a non-trigger collider — confirm with `get_component_info`.
- **`create_trigger_zone` is once-only by default** (tag filter + once flag) — re-entry won't re-fire unless you reset it. Good for one-shot goals, wrong for repeatable ones.
- **Two-phase:** scaffolds → recompile → THEN `component_add` + wire.
- **The marker needs a Canvas + the goal's transform** — no canvas, nothing renders; confirm with a `screenshot_game`.
- **`OnAllComplete` fires once** — hook the win/transition there, not in an `Update` poll.

## Cross-links

`references/systems/dialogue.md` (quest-givers), `references/systems/ui-hud.md` (objective list + markers), `references/systems/spawning-waves.md` / `references/systems/health-combat.md` (kill-N goals). Genres: `references/genres/puzzle.md`, `references/genres/platformer.md`, `references/genres/rpg.md`.
