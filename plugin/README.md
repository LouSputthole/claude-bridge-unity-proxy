# claude-bridge-unity — Claude Code Plugin

The "plugin with a brain" for building Unity games by talking to Claude.

This plugin bundles the **Claude Bridge for Unity** MCP server registration plus the skills that turn raw tool access into a disciplined game-dev workflow.

| Component | What it does |
|---|---|
| **MCP server registration** (`claude-bridge-unity`) | Points Claude Code at the Claude Bridge **proxy** — a self-contained stdio MCP server that drives the Unity Editor over a private WebSocket and **survives Unity domain reloads** (a normal recompile never breaks the session). Today it exposes GameObject / transform / component / scene / play-mode simulation / reflection-grounded inspection tools; higher-level authoring, gameplay-scaffold, NPC, and worldgen waves build on top. |
| **Skill: `claude-bridge-setup`** | A ~30-second onboarding wizard — verifies the bridge (`bridge_status`), reports the project + Unity version, recommends a first verifiable move, and points to help. |
| **Skill: `claude-bridge-build-feature`** | The soul of the plugin: the **screenshot-driven iteration workflow** (make a change → aim a screenshot → READ the PNG → fix) plus the hard-won Unity gotchas baked in — flat `"x,y,z"` string params, verify-the-mutation, reflection-first via `describe_type`, **compile-verify generated code** (`editor_recompile` → `wait_ready` → `get_compile_errors`), the **two-phase rule** for generated components, play-mode verification, and **handing input "feel" to a human**. Prevents the guess-and-check loop. |
| **Skill: `claude-bridge-api`** | Idiomatic **Unity 6 / C#** knowledge — MonoBehaviour lifecycle, `[SerializeField]` serialization rules, Editor vs runtime, `AssetDatabase`, coroutines, prefabs, physics — plus the **s&box / Unreal → Unity translation table** so the agent writes Unity, not another engine's patterns. Repointed to verify exact signatures via the bridge's live `describe_type`. |
| **Skill: `claude-bridge-cookbook`** | A concise **router** of Unity genre + system build orders (tycoon, survival, FPS, platformer, RPG, horror, racing, tower-defense, puzzle…; inventory, save, economy, health, NPC, spawning, dialogue, day/night, objectives…). Grows as the gameplay scaffolds ship. |
| **Skill: `claude-bridge-scaffold-game`** | One ask → a **playable first-person starter** you can press Play on. Orchestrates the gameplay scaffolds (`create_player_controller`, `create_game_manager`, `create_pickup`…) + primitives + colliders + an eye camera + `set_component_reference` wiring through the generate→recompile→place sequencing, then verifies with screenshots. Sibling of `claude-bridge-build-feature`. |
| **Agent: `unity-game-dev`** | A specialist **subagent** for handing off a self-contained Unity build task — it runs the screenshot-driven workflow end to end (brainstorm → API-research → implement → compile-verify → screenshot+read), and knows the two-phase rule, async play-mode polling, and to hand input "feel" to a human. |

## What this plugin does NOT include

This plugin gives Claude the **MCP client wiring + the brain (skills)**. To actually drive the Unity Editor you also need **two things installed separately**, and they work together:

1. **The proxy binary** — `claude-bridge-unity.exe`, the self-contained stdio MCP server. Built from `proxy/ClaudeBridge.Proxy` in the main repo.
2. **The Unity package** — `com.lousputthole.claudebridge` (Editor-only). It runs the WebSocket server inside the Unity Editor and writes a status file the proxy discovers.

### Install the proxy (one-time build)

Requires the .NET 8 SDK. From the repo root:

```bash
cd proxy
dotnet publish ClaudeBridge.Proxy/ClaudeBridge.Proxy.csproj \
  -c Release -r win-x64 -p:PublishSingleFile=true --self-contained true \
  -o publish/win-x64
# → publish/win-x64/claude-bridge-unity.exe
```

(macOS/Linux later: swap `-r osx-arm64` / `-r linux-x64`.)

Then make the plugin's `.mcp.json` find it. The bundled `.mcp.json` points at `${CLAUDE_PLUGIN_ROOT}/bin/win-x64/claude-bridge-unity.exe` (where `${CLAUDE_PLUGIN_ROOT}` is the installed plugin directory). You have two options:

- **Copy** the published `claude-bridge-unity.exe` into the plugin's `bin/win-x64/` folder (matches the default config), **or**
- **Point** the config at wherever you keep the binary by editing `.mcp.json`'s `command` to an absolute path, e.g. `C:/Users/you/Desktop/Claude-Bridge-Unity/proxy/publish/win-x64/claude-bridge-unity.exe`.

If you keep several Unity projects open, pin one by adding an env var:

```json
{
  "mcpServers": {
    "claude-bridge-unity": {
      "command": "${CLAUDE_PLUGIN_ROOT}/bin/win-x64/claude-bridge-unity.exe",
      "env": { "CLAUDE_BRIDGE_PROJECT": "C:/Games/GraveholdUnity" }
    }
  }
}
```

### Install the Unity package (per project)

Add it to your Unity project's `Packages/manifest.json` as a local file reference:

```json
{
  "dependencies": {
    "com.lousputthole.claudebridge": "file:../../path/to/Claude-Bridge-Unity/unity/com.lousputthole.claudebridge"
  }
}
```

(Or distribute it as a git URL / tarball / `.unitypackage` once published.) The package pulls in `com.unity.nuget.newtonsoft-json`. Open the project in Unity; on load the bridge starts its WebSocket server and writes `~/.claude-bridge/unity-<projecthash>.json`, which the proxy uses to find the port.

## Install the plugin

Once Claude Code's plugin marketplace catalogs this entry:

```
/plugin marketplace add LouSputthole/Claude-Bridge-Unity
/plugin install claude-bridge-unity
```

For local development you can point Claude at the plugin directory directly:

```
claude --plugin-dir /path/to/Claude-Bridge-Unity/plugin
```

**Restart your Claude Code session after installing or changing `.mcp.json`** — MCP servers load at startup, so the proxy only attaches on a fresh session.

## Verify it's working

In a new Claude Code session, ask:

```
Check the bridge status.
```

Claude should invoke `mcp__claude-bridge-unity__bridge_status` and report `bridgeVersion`, `toolCount`, `unityVersion`, and `projectName` — meaning the proxy reached a running Unity with the package installed. `mcp__claude-bridge-unity__ping` is the lightest liveness check.

- **"tool not found":** the MCP server isn't registered — confirm `.mcp.json`'s `command` resolves to the real exe, then restart Claude Code.
- **Times out / connection refused:** Unity isn't running, the package isn't in the project, or Unity is mid-compile. Open the project and retry (the proxy stays up and answers MCP even while Unity is down, then connects when Unity appears).

## Using the skills

The skills activate automatically when relevant, or invoke them explicitly:

```
/claude-bridge-unity:claude-bridge-setup            # first-run onboarding
/claude-bridge-unity:claude-bridge-build-feature    # the screenshot-driven build/verify workflow
/claude-bridge-unity:claude-bridge-scaffold-game    # one ask → a playable first-person starter
/claude-bridge-unity:claude-bridge-api              # idiomatic Unity C#
/claude-bridge-unity:claude-bridge-cookbook         # genre/system build orders
```

For a focused hand-off, the **`unity-game-dev`** agent runs this whole workflow as a subagent — give it a self-contained task ("build a wave-spawner system", "lay out and light the test level") and it brainstorms, researches the API, implements, compile-verifies, and screenshots its own work.

`claude-bridge-build-feature` enforces the disciplines that make the bridge productive:
1. Confirm the bridge is alive before doing anything.
2. Brainstorm complex features before coding.
3. Research the Unity API via `describe_type` before guessing (reflection-first).
4. Bite-sized edits, one file at a time, writes confined to the project.
5. **Compile-verify** generated code (`editor_recompile` → `wait_ready` → `get_compile_errors`) and obey the **two-phase rule**.
6. **Screenshot + read it yourself** for any visual change; **verify the mutation took** for any state change.

…plus the Unity gotchas (flat `"x,y,z"` params, fake-null on destroyed objects, `FindObjectsByType`/`linearVelocity` renames, render-pipeline material mismatches, editor-vs-runtime API, tags/layers must pre-exist) and the honest limit that **the bridge cannot synthesize input "feel"** — that goes to a human playtester.

## What's bundled vs. installed separately

- The **skills** and **MCP registration** are bundled with this plugin.
- The **proxy binary** is built from the repo and placed/pointed-at via `.mcp.json` (not bundled prebuilt).
- The **Unity package** (the editor-side C#) installs into your Unity project (not bundled).

## Version compatibility

This plugin, the proxy, and the Unity package are all **v0.1.0** (early development). Keep them in lockstep — the proxy and package share the wire contract and `bridge_status` reports the version each side runs. If you upgrade one, upgrade all three.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `mcp__claude-bridge-unity__*` tools not available | Plugin not installed, or session not restarted | Reinstall, restart Claude Code (MCP loads at startup) |
| Tools time out | Unity not running, package not in project, or mid-compile/reload | Open the Unity project; wait for compilation to finish and retry |
| `command` not found at startup | `.mcp.json` path doesn't resolve to the exe | Build the proxy and copy it to `bin/win-x64/`, or set `command` to its absolute path |
| Connects to the wrong Unity instance | Several editors open | Set `CLAUDE_BRIDGE_PROJECT` to the project path in `.mcp.json`'s `env` |
| A generated component won't attach | Skipped the two-phase rule | `editor_recompile` → `wait_ready` → `get_compile_errors` (confirm clean) → then `component_add` |

## License

Proprietary, commercial. See the repo's `LICENSE` and `THIRD-PARTY-NOTICES.md`. The Claude Bridge for Unity is original work; the realvirtual Unity MCP and the maintainer's own s&box Claude Bridge were studied as references only — no third-party source copied.

## Credits

Built by **LouSputthole**. The skill set mirrors the structure of the maintainer's s&box Claude Bridge plugin, adapted to Unity 6 / C# and this bridge's tools.
