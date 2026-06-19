---
name: claude-bridge-api
description: Use when writing or modifying C# for a Unity project driven through the Claude Bridge — MonoBehaviour components, ScriptableObjects, Editor scripts, and gameplay code. Teaches idiomatic Unity 6 / C# patterns (MonoBehaviour lifecycle, [SerializeField] serialization, Editor vs runtime, AssetDatabase, coroutines, prefabs, physics) and the s&box/Unreal -> Unity translations, so the agent writes Unity, not another engine's patterns. Verify exact signatures live via the bridge's describe_type.
---

# Claude Bridge — Unity C# Skill

## READ BEFORE WRITING CODE

**Unity is not s&box, and not Unreal.** If you came from the maintainer's s&box bridge, your muscle memory is wrong here. In Unity there is no `Component` base with `OnUpdate()`, no `[Property]` attribute, no `[Sync]`, no `Scene.Trace` builder, no `WorldPosition` on the component, no `prefab.Clone()`, no Razor UI, no Z-up. Unity has its own lifecycle, serialization model, and API surface. If you write an s&box-ism (or a Unreal-ism), you've hallucinated.

Unity is a C# scripting layer over the Unity engine. Gameplay code is **`MonoBehaviour`** subclasses attached to **`GameObject`**s; data lives in **`ScriptableObject`**s; editor tooling lives in **`Editor`**-assembly scripts. **The single source of truth is the LIVE editor reflection via the Claude Bridge** — `mcp__claude-bridge-unity__describe_type` / `search_types` / `get_method_signature` query your actually-installed Unity version + packages. If anything here disagrees with what the bridge reports for your build, the live reflection wins.

**This skill is the *brain* — how to write correct Unity C#. To build, run, and SEE it, pair it with the `claude-bridge-build-feature` skill** (the screenshot-driven editor workflow + the verify-the-mutation / two-phase / play-mode disciplines). Write it right here; verify it live there.

---

## Architecture in 30 seconds

```
Scene  ──>  GameObject (Transform is mandatory; has tags, layer, children, components)
                  └── MonoBehaviour (gameplay code)        ← Component subclass, attached in scene/prefab
            ScriptableObject (data asset, lives in Assets/, not in the scene)
            Editor script (Assembly "Editor" only; AssetDatabase, custom inspectors, menus)
```

- **All gameplay code** is a `class Foo : MonoBehaviour`. Attach it to a GameObject.
- **Lifecycle methods are magic-named** (Unity calls them by name via reflection): `Awake()`, `OnEnable()`, `Start()`, `Update()`, `FixedUpdate()`, `LateUpdate()`, `OnDisable()`, `OnDestroy()`. They are `private void` by convention — **not** `override` (MonoBehaviour does not declare them virtual).
- **Transforms** are on the GameObject's `Transform` component, reached via `transform.position` / `transform.localPosition` / `transform.rotation` from any MonoBehaviour.
- **Serialize a field to the inspector** with `[SerializeField] private T x;` (or a `public T x;`, but `[SerializeField] private` is the idiom). Only serializable types persist (see the serialization rules below).
- **Data assets** are `ScriptableObject`s — create with `[CreateAssetMenu]`, instantiate in the project, reference from components.
- **Coordinate system is Y-up, left-handed:** `Vector3.forward = (0,0,1)`, `Vector3.up = (0,1,0)`, `Vector3.right = (1,0,0)`.
- **Physics:** `Rigidbody` + `Collider` components; raycasts are `Physics.Raycast(origin, dir, out hit, maxDist, layerMask)`; collisions arrive via `OnCollisionEnter(Collision)` / `OnTriggerEnter(Collider)` magic methods (the body needs a Rigidbody for these to fire).
- **Async:** Unity uses **coroutines** (`IEnumerator` + `StartCoroutine`, `yield return`), or plain `async/await` / `Awaitable` (Unity 6). `Task` works but be careful — it doesn't auto-cancel on object destroy.

---

## Routing table — "I need to…"

Match the task. For exact signatures on your build, always finish with the bridge's `describe_type` / `get_method_signature`.

| Task | Approach (verify live with the bridge) |
|---|---|
| Write a component | `class Foo : MonoBehaviour`; lifecycle in `Awake/OnEnable/Start/Update/FixedUpdate/OnDisable/OnDestroy` |
| Expose a field to the inspector | `[SerializeField] private float speed = 5f;` (or `public`); use `[Range]`, `[Tooltip]`, `[Header]` |
| Reference another object/component | `[SerializeField] private Transform target;` and wire it in the inspector (or via the bridge) |
| Find components/objects at runtime | `GetComponent<T>()`, `GetComponentInChildren<T>()`, `FindObjectsByType<T>(...)` (NOT the old `FindObjectsOfType` in Unity 6) |
| Create/destroy/clone | `Instantiate(prefab, pos, rot)`, `Destroy(go)` (or `DestroyImmediate` in editor code only) |
| A data asset (config, item, recipe) | `class Item : ScriptableObject` + `[CreateAssetMenu(menuName="…")]` |
| Timed/sequenced logic | coroutine (`IEnumerator`, `yield return new WaitForSeconds(t)`) or `async`/`Awaitable` |
| Physics movement | move a `Rigidbody` in `FixedUpdate` (`MovePosition`/`AddForce`/set `linearVelocity`) |
| Raycast / overlap | `Physics.Raycast(...)`, `Physics.OverlapSphere(...)`, `Physics.SphereCast(...)` with a `LayerMask` |
| Collisions/triggers | implement `OnCollisionEnter(Collision)` / `OnTriggerEnter(Collider)` (collider + Rigidbody required) |
| Input | the **Input System** package (`InputAction`/`PlayerInput`) is current; legacy `Input.GetKey/GetAxis` still exists — check which the project uses |
| UI | **UI Toolkit** (`.uxml`/`.uss`, `UIDocument`) for new work, or **uGUI** (`Canvas`/`Image`/`Text`/`TMP`) — check which the project uses |
| Editor tooling (menus, inspectors, asset ops) | an **Editor**-assembly script using `UnityEditor.*` — wrap usages in `#if UNITY_EDITOR` if in a runtime file |
| Load/save/create assets (editor) | `AssetDatabase.CreateAsset/LoadAssetAtPath/Refresh/SaveAssets` — **editor only**, never at runtime |
| Persist game data at runtime | `Application.persistentDataPath` + JSON (`JsonUtility`) / a binary writer — NOT `AssetDatabase` |
| Exact signature of any Unity type | **Claude Bridge (live):** `describe_type` / `get_method_signature` / `list_components` |

---

## s&box / Unreal → Unity translation table

If you write the left column in a Unity project, you've hallucinated. Use the right column.

| s&box / Unreal / wrong | Unity / correct |
|---|---|
| `class Foo : Component` (s&box) / `: Actor`/`: UObject` (Unreal) | `class Foo : MonoBehaviour` |
| `protected override void OnUpdate()` (s&box) / `Tick()` (Unreal) | `void Update()` (magic method, not override) |
| `protected override void OnStart()` | `void Start()` |
| `protected override void OnAwake()` | `void Awake()` |
| `protected override void OnFixedUpdate()` | `void FixedUpdate()` |
| `protected override void OnEnabled()` / `OnDisabled()` | `void OnEnable()` / `void OnDisable()` |
| `protected override void OnDestroy()` | `void OnDestroy()` (same name, but NOT override) |
| `[Property] public float Speed { get; set; }` (s&box) / `UPROPERTY()` (Unreal) | `[SerializeField] private float speed;` |
| `[Hide]` | `[HideInInspector]` |
| `WorldPosition` / `LocalPosition` (on the component) | `transform.position` / `transform.localPosition` |
| `WorldRotation` | `transform.rotation` (a `Quaternion`); euler via `transform.eulerAngles` |
| `GameObject.Enabled = false` | `gameObject.SetActive(false)` |
| `GameObject.Destroy()` / `prefab.Clone(pos)` | `Destroy(gameObject)` / `Instantiate(prefab, pos, rot)` |
| `Scene.Get<T>()` / `Scene.GetAll<T>()` (s&box) | `FindFirstObjectByType<T>()` / `FindObjectsByType<T>(FindObjectsSortMode.None)` (Unity 6) |
| `Scene.Directory.FindByName("X")` | `GameObject.Find("X")` (avoid in hot paths; prefer a serialized ref) |
| `Scene.Trace.Ray(a,b).Run()` (builder) | `Physics.Raycast(origin, direction, out RaycastHit hit, maxDistance, layerMask)` |
| `Component.ICollisionListener.OnCollisionStart(c)` | `void OnCollisionEnter(Collision c)` (magic method) |
| `Component.ITriggerListener.OnTriggerEnter(c)` | `void OnTriggerEnter(Collider other)` (magic method) |
| `Rigidbody.ApplyImpulse(f)` / `Rigidbody.Velocity` | `rb.AddForce(f, ForceMode.Impulse)` / `rb.linearVelocity` (Unity 6 renamed `velocity`→`linearVelocity`) |
| `Input.Down("forward")` / `Input.Pressed("x")` (s&box actions) | legacy: `Input.GetKey/GetKeyDown`; new: an `InputAction` (`action.IsPressed()` / `action.WasPressedThisFrame()`) |
| `Input.AnalogMove` | `Vector2 move = moveAction.ReadValue<Vector2>();` (Input System) or `Input.GetAxis` (legacy) |
| `Scene.Camera` | `Camera.main` (the camera tagged MainCamera) |
| `async Task Foo()` + `await Task.DelaySeconds(n)` (s&box) | a coroutine: `IEnumerator Foo(){ yield return new WaitForSeconds(n); }` + `StartCoroutine(Foo())`; or `await Awaitable.WaitForSecondsAsync(n)` (Unity 6) |
| `await Task.Frame()` | `yield return null;` (coroutine) |
| `Log.Info(x)` / `Log.Warning` / `Log.Error` | `Debug.Log(x)` / `Debug.LogWarning(x)` / `Debug.LogError(x)` |
| `Time.Now` / `Time.Delta` | `Time.time` / `Time.deltaTime` (fixed step: `Time.fixedDeltaTime`) |
| `MathX.Lerp / Clamp` | `Mathf.Lerp / Mathf.Clamp` |
| `Game.Random.Next(a,b)` | `Random.Range(a,b)` (`UnityEngine.Random`; int overload is max-exclusive, float overload max-inclusive) |
| `Vector3.Forward = (1,0,0)` (s&box Z-up) | `Vector3.forward = (0,0,1)` — **Unity is Y-up**; re-check every literal direction |
| `Scene.LoadFromFile(...)` | `SceneManager.LoadScene("Name"|index)` (namespace `UnityEngine.SceneManagement`) |
| `go.Flags = GameObjectFlags.DontDestroyOnLoad` | `DontDestroyOnLoad(gameObject)` |
| `Game.IsPlaying` | `Application.isPlaying`; in editor code, `EditorApplication.isPlayingOrWillChangePlaymode` |
| `FileSystem.Data.ReadAllText(...)` (s&box sandbox) | `System.IO.File.*` under `Application.persistentDataPath` (full BCL IO is allowed — Unity is not sandboxed) |
| `[Sync]` / `[Rpc.Broadcast]` (s&box netcode) | Unity has no built-in equivalent; use **Netcode for GameObjects** (`NetworkBehaviour`, `NetworkVariable<T>`, `[Rpc]`) or another netcode package — verify the project's choice |

If a pattern isn't here, assume it differs and verify with `describe_type` before writing it.

---

## The rules you must not break

1. **Gameplay classes extend `MonoBehaviour`; data classes extend `ScriptableObject`.** Not `Component` (that's s&box / the Unity base you don't subclass directly), not `Actor`.
2. **Lifecycle methods are magic-named, not `override`.** `void Update()` — Unity finds it by name. Writing `override` won't compile (they aren't virtual); writing `OnUpdate()` does nothing.
3. **`Update` for per-frame logic, `FixedUpdate` for physics.** Read input in `Update`; move/force a `Rigidbody` in `FixedUpdate`. Multiply per-frame rates by `Time.deltaTime`.
4. **Serialize with `[SerializeField]`.** A `private` field without it is NOT saved or shown. Only Unity-serializable types persist: primitives, `string`, enums, `UnityEngine.Object` references, `Vector*/Color/Quaternion`, and `[System.Serializable]` plain classes/structs — **not** `Dictionary`, interfaces, or plain `object` (use `[SerializeReference]` for polymorphism, or parallel lists for a dict).
5. **Wire references in the inspector, don't `Find` in hot paths.** `[SerializeField] private Foo foo;` set once beats `GameObject.Find`/`GetComponent` every frame.
6. **`UnityEditor` is editor-only.** Never call `AssetDatabase`, `EditorUtility`, `[MenuItem]`, custom inspectors, etc. from runtime code. If a runtime file must, guard with `#if UNITY_EDITOR … #endif`. Editor scripts live in an `Editor/` folder or an asmdef marked Editor.
7. **`AssetDatabase` is for editor assets; `persistentDataPath` is for player saves.** Don't try to write game saves via `AssetDatabase` — it doesn't exist in a build.
8. **Check for "fake null."** A destroyed `UnityEngine.Object` compares `== null` (overloaded `==`) even though the C# reference isn't null. Always null-check serialized refs before use; don't cache a reference across a `Destroy`.
9. **Coroutines need an active MonoBehaviour.** `StartCoroutine` stops if the component/GameObject is disabled or destroyed. For lifetime-safe timed work prefer coroutines over raw `Task` (a `Task` keeps running after the object dies and swallows exceptions).
10. **Look up every API before you use it — live.** `describe_type` (members), `search_types` (does it exist?), `get_method_signature` (exact overloads + param names). Unity's API shifts across versions, render pipelines, and packages; live reflection beats any static list. Several APIs were renamed in recent versions (`velocity`→`linearVelocity`, `FindObjectsOfType`→`FindObjectsByType`) — don't assume.

---

## Project structure (Unity)

```
MyGame/
├── Assets/
│   ├── Scenes/            *.unity
│   ├── Scripts/           gameplay *.cs (compiled into Assembly-CSharp unless an asmdef scopes them)
│   │   └── Editor/        editor-only *.cs (or an asmdef with Editor platform only)
│   ├── Prefabs/           *.prefab
│   ├── Materials/         *.mat
│   ├── Models/  Textures/  Audio/  …
│   └── Resources/         loadable at runtime via Resources.Load (use sparingly; Addressables preferred)
├── Packages/
│   └── manifest.json      UPM dependencies (this is where a local file: package ref goes)
├── ProjectSettings/       render pipeline, tags & layers, input, physics
└── Library/  Temp/  Logs/ (generated — never edit)
```

Notable:
- **Every asset has a `.meta`** with a stable GUID — references survive renames. Don't delete `.meta` files; let Unity manage them.
- C# under `Assets/` compiles into `Assembly-CSharp` by default; an **`.asmdef`** scopes a folder into its own assembly (faster compiles, explicit deps, an Editor platform flag).
- Asset paths in editor APIs are project-relative and start with `Assets/`: `AssetDatabase.LoadAssetAtPath<GameObject>("Assets/Prefabs/Player.prefab")`.
- The **render pipeline** (Built-in / URP / HDRP) is a project-wide choice in ProjectSettings — materials and shaders are pipeline-specific.

---

## A reference component (shape only — not the content)

The *shape* of an idiomatic Unity component. The *content* depends on what you're building — verify the APIs live, then write it.

```csharp
using UnityEngine;

[DisallowMultipleComponent]
public class Mover : MonoBehaviour
{
    [Header("Tuning")]
    [SerializeField, Range(0f, 20f)] private float speed = 6f;
    [SerializeField, Tooltip("Wired in the inspector or via the bridge")]
    private Transform target;

    private Rigidbody _rb;          // cached in Awake; '_' private-field convention

    private void Awake()
    {
        _rb = GetComponent<Rigidbody>();   // cache component refs once
    }

    private void Update()
    {
        // per-frame, non-physics: input, timers, visuals
        if (target == null) return;        // guard fake-null / unwired refs
    }

    private void FixedUpdate()
    {
        // physics: move the Rigidbody here, scaled by the fixed step
        if (_rb == null || target == null) return;
        Vector3 dir = (target.position - _rb.position).normalized;
        _rb.MovePosition(_rb.position + dir * speed * Time.fixedDeltaTime);
    }

    private void OnTriggerEnter(Collider other)
    {
        // collider + Rigidbody required for this to fire
    }
}
```

And a data asset:

```csharp
using UnityEngine;

[CreateAssetMenu(menuName = "MyGame/Item")]
public class ItemDef : ScriptableObject
{
    public string displayName;
    public Sprite icon;
    public int maxStack = 99;
}
```

---

## Gotchas captured from real Unity builds

- **`FindObjectsOfType` is obsolete in Unity 6** — use `FindObjectsByType<T>(FindObjectsSortMode.None)` (and `FindFirstObjectByType<T>()`). The old names warn or error depending on version.
- **`Rigidbody.velocity` → `linearVelocity`, `angularVelocity` stays** (Unity 6 rename). `describe_type "Rigidbody"` to confirm on your build.
- **A `[SerializeField] private` field already present in the scene/prefab keeps its saved value** — changing the initializer in code does NOT change existing instances. Reset in the inspector or set via the bridge.
- **`Awake` runs on load even if the component is disabled; `Start` waits for the first frame the component is enabled; `OnEnable` runs every enable.** Order: `Awake` (all) → `OnEnable` → `Start` → `Update`s.
- **Execution order across objects is undefined** unless you set Script Execution Order — don't rely on one object's `Awake` running before another's.
- **`Destroy` is deferred** to end of frame; `DestroyImmediate` is immediate but **editor-only / dangerous at runtime**. The object survives the rest of the current frame after `Destroy`.
- **In code reachable from EDIT mode, bare `Destroy(o)` silently no-ops AND logs a warning** — it only acts in play mode. A bootstrapper that builds the scene in `Awake` is very often *also* run as an edit-mode bake (a `[ContextMenu]` "bake", `[ExecuteAlways]`, or an editor script), and bare `Destroy` there leaves the very objects (colliders, temp props) it meant to strip. Use the play-mode-aware form in any teardown reachable from the editor: `if (Application.isPlaying) Destroy(o); else DestroyImmediate(o);`. (Pure play-mode paths — pickups, despawns, singleton dedup — stay plain `Destroy`.)
- **`GetComponent` returns C#-`null` (not fake-null) when absent**, but a destroyed component is fake-null — handle both with a plain `== null` check (Unity's operator covers both).
- **`OnCollision*`/`OnTrigger*` need a Rigidbody on at least one of the two bodies**, and the right `isTrigger` settings — a missing Rigidbody is the #1 reason a trigger "doesn't fire."
- **Coroutine `yield return new WaitForSeconds(t)` uses scaled time** (stops when `Time.timeScale == 0`); use `WaitForSecondsRealtime` for unscaled.
- **`[SerializeField]` won't serialize a `Dictionary`, an `interface`, or a property** — use parallel `List`s, `[SerializeReference]`, or a `[System.Serializable]` wrapper.
- **`Camera.main` does a tag search** — cache it; don't call it every frame.
- **Material instancing leak:** reading `renderer.material` clones the material (a per-renderer instance you must clean up); read `renderer.sharedMaterial` if you don't intend to instance.
- **`Resources.Load` and a `Resources/` folder ship everything in it, always loaded** — prefer **Addressables** for anything non-trivial; verify which the project uses.
- **Editor API in a runtime file breaks the build** — `#if UNITY_EDITOR` guard or move it to an Editor asmdef.

---

## Verification loop (when in doubt)

1. **Check this skill** for the pattern + the translation away from s&box/Unreal.
2. **Verify it live with the bridge.** `search_types query="Foo"` to find it, `describe_type typeName="Foo"` for members, `get_method_signature` for exact overloads. This reflects YOUR Unity + packages — the real source of truth.
3. **If reflection doesn't show it, it doesn't exist** in your build (or it's in a package you haven't imported). Don't write it — find the Unity-idiomatic way.
4. **For prose/manual docs** (render pipeline, Input System, a package), use WebFetch on the Unity Manual / Scripting API or the `context7` MCP.

This skill teaches the patterns + mental model; the bridge's live reflection is the authoritative signature check. Then go build/run/SEE it with `claude-bridge-build-feature`.
