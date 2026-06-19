# Health / Damage / Combat

A health pool with damage/heal/death events, and the weapon side (hitscan via `raycast`, or projectile prefabs) that drives it. Built on the real `create_health_system` scaffold.

## What it IS / when you need it

A `Health` Component with `current`/`max`, `TakeDamage`/`Heal`/`Revive`, `IsDead`/`HealthFraction`, and `OnDamaged`/`OnHealed`/`OnDeath` events. Put it on anything mortal — player, enemy, destructible prop. Weapons/hazards call `TakeDamage`; they don't need to know what they hit.

## Bridge build order

**Phase A — generate + compile:**
1. `create_health_system name="Health" maxHealth=100` — generates the Health MonoBehaviour with the methods + events above.
2. (Optional, for stats-driven games) `create_stat_modifier_system name="CombatStats" statsCsv="Health,Damage,Armor"` so max-health/damage come from modifiable stats — see `references/systems/progression-upgrades.md`.
3. `editor_recompile` → `wait_ready` → `get_compile_errors`.

**Phase B — place + wire:**
4. `component_add target="Player" type="Health"` (and on each enemy prefab).
5. Confirm: `get_component_fields target="Player" componentType="Health"` → max/current.
6. **Hitscan weapon:** a small `create_script` weapon that, on fire, casts a ray. Use the **runtime** `raycast origin="<muzzle>" direction="<forward>" maxDistance=100` to prototype/aim and read what it hit, then in your weapon code call `Physics.Raycast` and `TakeDamage` on the hit's `Health`. Verify the geometry with the bridge `raycast` tool first — it returns the hit path/point/normal/distance, so you can confirm your muzzle/forward math before writing the loop.
7. **Projectile weapon:** a projectile prefab with a Rigidbody (`physics_add_rigidbody`) + collider; on collision it calls `TakeDamage`. Spawn via `prefab_instantiate` or a `create_spawner`.
8. **Death:** subscribe to `OnDeath` to ragdoll/despawn/score. For enemies, that's where the loot roll fires (`create_weighted_loot_table`).

## Verify

- **Edit-scene:** `get_component_fields` confirms the pool.
- **Runtime, the unambiguous proof:** `sim_play` → poll `sim_status`. `invoke_method target="Player" componentType="Health" methodName="TakeDamage" argsJson="[30]"` → `get_runtime_property … propertyPath="HealthFraction"` ≈ 0.7. Drive it to death (`argsJson="[100]"`) and read `IsDead` → true; confirm `OnDeath` fired by observing the result (despawn, score change). `Heal`/`Revive` the same way. This is far more reliable than trying to shoot in play mode — the bridge can't aim a mouse.
- **Weapon aim:** the bridge `raycast` tool lets you confirm a shot from the muzzle actually reaches the target before you trust the in-code cast.

## Variations

- **Armor / resistances:** route `TakeDamage` through a stat (`create_stat_modifier_system`) so buffs/gear modify incoming damage.
- **Invulnerability frames / regen:** add a cooldown timer or a slow `Heal` tick to the health script (`edit_script`), recompile.
- **Destructibles:** the same `Health` on a prop; `OnDeath` swaps the model or spawns debris.
- **Damage over time** (fire/poison): a separate component that calls `TakeDamage` on a `Time.deltaTime`-scaled or fixed-interval tick.

## Gotchas

- **`TakeDamage` should clamp to `[0,max]`** (the scaffold does) — don't let health go negative or over max; verify with the fraction read-back.
- **Two-phase:** `create_health_system` → recompile → THEN `component_add` on player and enemies.
- **Hitscan needs colliders on targets** — a `raycast` only hits things with colliders. Confirm with `get_component_info` on the target. The bridge `raycast` returning `hit:false` usually means a missing collider or wrong direction, not a wrong damage value.
- **Move projectiles via the Rigidbody, fire logic in `Update`, collisions resolve in `FixedUpdate`** — raw transform-setting a projectile skips collision.
- **The bridge can't judge weapon *feel*** (recoil, fire rate, hit-stop) — wire the logic, verify the numbers, then hand the human a playtest.

## Cross-links

`references/systems/npc-ai.md` (enemies that attack), `references/systems/spawning-waves.md` (where enemies come from), `references/systems/progression-upgrades.md` (damage/armor as stats), `references/systems/ui-hud.md` (health bar). Genres: `references/genres/fps-shooter.md`, `references/genres/top-down-twin-stick.md`, `references/genres/survival.md`.
