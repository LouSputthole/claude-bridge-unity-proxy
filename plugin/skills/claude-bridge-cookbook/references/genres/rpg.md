# RPG — Genre Recipe

A character with stats that grow, an inventory of gear, quests to follow, loot to find, and the whole thing persists. Built from the stat-modifier, inventory, objective, loot, dialogue, and save scaffolds — the most system-dense genre.

## The loop

Explore → talk to NPCs (get quests) → fight enemies / solve tasks → earn XP + loot → level up stats and equip gear → advance quests → save. It's a web of systems more than a single tight loop.

## The system stack to compose

1. **Character stats** — `create_stat_modifier_system name="CharStats" statsCsv="Health,Mana,Strength,Damage,Defense"` (`references/systems/progression-upgrades.md`). The hub everything reads.
2. **Health/combat** — `create_health_system` (max from the Health stat), enemies too (`references/systems/health-combat.md`).
3. **Inventory + equipment** — `create_inventory` + per-instance gear (`references/systems/inventory.md`); equipping adds stat modifiers.
4. **Loot** — `create_weighted_loot_table` on enemy death / chests; `create_pickup` for drops.
5. **Quests** — `create_objective_system` + `create_objective_marker` (`references/systems/objectives-quests.md`).
6. **Dialogue** — `create_interactable` + a dialogue runner (`references/systems/dialogue.md`); choices give/advance quests.
7. **Save** — `create_save_system` persisting stats/levels, inventory, quest flags (`references/systems/save-load.md`).
8. **NPCs/enemies** — `create_npc_brain` (friendly `wanderer` or hostile `predator`) (`references/systems/npc-ai.md`).
9. **HUD** — health/mana bars, quest log, inventory screen (`references/systems/ui-hud.md`).

## Bridge build order

Build **one vertical slice first** (player can move, talk to one NPC, get one quest, kill one enemy, get loot, level up, save) before fanning out.

**1. Block out a small area.** Ground + a town stand-in + an enemy spot. `bake_navmesh`. `screenshot_scene`.

**2. Phase A — generate the slice, then compile:**
- `create_player_controller name="Hero" mode="third_person"`
- `create_stat_modifier_system name="CharStats" statsCsv="Health,Mana,Strength,Damage,Defense"`
- `create_health_system name="Health" maxHealth=100`
- `create_inventory name="Bag" slots=24`
- `create_objective_system name="Quests"`, `create_objective_marker name="QuestMarker"`
- `create_interactable name="NpcTalk" prompt="Press E to talk" range=3` + a dialogue runner script
- `create_npc_brain name="Enemy" preset="predator"`, `create_weighted_loot_table name="Loot"`, `create_pickup name="Drop" kind="gold"`
- `create_save_system name="SaveSystem"`
- `editor_recompile` → `wait_ready` → `get_compile_errors`.

**3. Phase B — place + wire the slice:**
- Hero: `component_add` Hero + Health + CharStats + Bag; Health's max reads `CharStats.Get(Health)`.
- NPC: `NpcTalk` + dialogue runner; a dialogue choice registers a quest in `Quests`; `QuestMarker` points at the objective.
- Enemy: `add_navmesh_agent` + `Enemy` brain + a Health; `OnDeath` → `Loot.Roll()` → spawn `Drop` + advance the kill quest.
- Equipping a piece of gear in `Bag` adds a modifier to `CharStats` (e.g. +Damage); unequip removes it by source.
- Save POCO: stats/levels + bag + quest flags; load clamps + re-applies gear modifiers.

## How to verify each step

- **Stats:** `sim_play`, `get_runtime_property target="Hero" componentType="CharStats" propertyPath="<Damage>"`; equip gear (`invoke_method` the equip / `AddModifier`) → value rises; unequip → reverts.
- **Dialogue → quest:** `invoke_method` the dialogue runner's `StartConversation`, `Choose(index)`, then `get_runtime_property` the `Quests` system to confirm the quest registered.
- **Combat → loot → quest advance:** `invoke_method TakeDamage` an enemy to death → confirm a `Drop` spawned (`find_in_play`) and the kill objective advanced.
- **Save round-trip:** change stats/bag/quests, Save, `sim_stop`, `sim_play`, Load, read all three back — they match, and gear modifiers re-applied (stat still buffed). This is the genre's hardest verify; do it explicitly.
- **HUD:** `screenshot_game`, read health/mana bars + quest log.

## Gotchas

- **Store *levels/owned-gear*, not resolved stats, in the save** — re-derive the resolved value on load (it's a function of levels + gear + the current balance table).
- **Re-apply equipment modifiers on load**, or loaded characters are weaker than when saved.
- **No `Dictionary` in serialized state** (quest flags, inventory) — use `[System.Serializable]` lists.
- **Quests need the player tagged + trigger goals wired** (see `references/systems/objectives-quests.md`).
- **This genre is the most system-coupled** — build and verify ONE slice end-to-end before adding the second quest/enemy/gear type, or you debug five half-wired systems at once.

## Cross-links

Systems: progression-upgrades, inventory, objectives-quests, dialogue, health-combat, npc-ai, save-load, ui-hud. Borrows combat from `references/genres/fps-shooter.md`/`top-down-twin-stick.md`. Workflow: `claude-bridge-build-feature`.
