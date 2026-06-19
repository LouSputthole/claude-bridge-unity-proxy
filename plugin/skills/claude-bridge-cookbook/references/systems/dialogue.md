# Dialogue / Conversation

Talk-to-an-NPC: an interactable that opens a conversation, advances lines, and branches. Built on the real `create_interactable` scaffold + a data-driven line graph + a uGUI panel.

## What it IS / when you need it

A conversation system = **data** (lines, speaker, optional choices/branches) + **a trigger** (walk up, press the interact key) + **a UI** (the dialogue box, advance, choice buttons) + **state** (which node you're on, what's been said). Reach for it for shopkeepers, quest-givers, story beats, tutorials.

## Bridge build order

**Phase A — generate + compile:**
1. `create_interactable name="DialogueTrigger" prompt="Press E to talk" range=3` — generates an `IInteractable` with a prompt, range, `CanInteract`, `Interact`, `OnInteract`. This is how the player starts the conversation.
2. Author the line data: a `[System.Serializable]` node type (id, speaker, text, next-id or a small list of choices) in a `create_script` file, and either an inline array or a `create_scriptable_object` conversation asset. **Use a list of structs with explicit next-ids — never a `Dictionary`** (won't serialize).
3. A `DialogueRunner` Component (`create_script`) that holds the current node, exposes `StartConversation`/`Advance`/`Choose(index)`, and fires an event when the line changes.
4. `editor_recompile` → `wait_ready` → `get_compile_errors`.

**Phase B — place + wire:**
5. NPC: `component_add target="NPC" type="DialogueTrigger"` + `component_add target="NPC" type="DialogueRunner"`; the trigger's `OnInteract` calls `runner.StartConversation(...)`.
6. UI: `create_canvas`, then a `create_ui_panel` dialogue box with a `create_ui_text` for the line and `create_ui_button`(s) for advance/choices. Bind the runner's line-changed event to `set_ui_text` and toggle the choice buttons. See `references/systems/ui-hud.md`.

## Verify

- **Edit-scene:** `get_component_fields target="NPC" componentType="DialogueTrigger"` → prompt + range. `screenshot_scene` after building the panel to confirm the box lays out on screen (read the PNG; uGUI anchoring is easy to get wrong).
- **Runtime:** `sim_play` → poll `sim_status`. Start the conversation directly: `invoke_method target="NPC" componentType="DialogueRunner" methodName="StartConversation" argsJson="[…]"`, then `get_runtime_property … propertyPath="<current line / node id>"`. `invoke_method … methodName="Advance"` and re-read — the node should move. For a branch, `Choose(0)` vs `Choose(1)` should land on different nodes. `screenshot_game` to see the box render the line. This drives the whole flow without needing to press the interact key.

## Variations

- **Linear vs branching:** linear = each node has one next-id; branching = a node carries a small list of `{choiceText, nextId}`. The runner exposes `Advance` for linear, `Choose(index)` for branches.
- **Conditions / flags:** gate a branch on a quest flag or inventory item — the runner checks before showing a choice. Ties into `references/systems/objectives-quests.md`.
- **Typewriter effect / portraits / voice:** cosmetic layers on the line-changed event (reveal text over time, swap a portrait Image, `play_sound`). Don't let cosmetics block `Advance`.
- **ScriptableObject conversations** let designers author dialogue as assets without recompiling — `create_scriptable_object` per conversation.

## Gotchas

- **No `Dictionary` in the node graph** — use a list of `[System.Serializable]` nodes with explicit ids, or `JsonUtility`/the inspector won't serialize it.
- **Two-phase:** `create_interactable` + your runner script → recompile → THEN `component_add` + wire.
- **The interact range/prompt is on the scaffold** — confirm `range` with `get_component_fields`; too small and the player can never trigger it.
- **UI not updating** usually means the line-changed event isn't firing or the label isn't bound — verify by driving `Advance` via `invoke_method` and reading the node, then check the label with a `screenshot_game`.
- **Block player movement/input during dialogue** if it shouldn't walk away mid-line — that's a game decision, wire it in the runner's start/end.

## Cross-links

`references/systems/ui-hud.md` (the dialogue box), `references/systems/objectives-quests.md` (dialogue that gives/advances quests), `references/systems/npc-ai.md` (the NPC you talk to). Genres: `references/genres/rpg.md`, `references/genres/tycoon-management.md` (shopkeepers).
