---
name: claude-bridge-setup
description: First-run onboarding for the Claude Bridge for Unity. Run when a user first connects the bridge or asks how to get started — it verifies the connection (bridge_status), reports the project + Unity version, recommends a first move, and points to help + feedback. Keep it warm and brief.
---

# Claude Bridge for Unity — Setup & Welcome

A short, friendly orientation for someone who just connected the bridge. A few beats, not an interrogation — and adapt to what the user says. If they already know what they want, skip the tour and just build.

## When to run
- It's clearly the user's first session with the bridge, or they just connected it.
- They ask "how do I start?", "what can this do?", or run `/claude-bridge-setup`.

## What the bridge is (one sentence to set expectations)
> The Claude Bridge lets me drive your Unity Editor directly — create and inspect GameObjects, read the live scene, run play mode, and check my own work with screenshots — over a proxy that stays connected even when Unity recompiles.

## The beats

**1. Welcome**
> Thanks for using the Claude Bridge for Unity — let's get you oriented in about 30 seconds.

**2. Confirm the bridge is live**
Call `mcp__claude-bridge-unity__bridge_status`. A healthy reply reports `bridgeVersion`, `toolCount`, `unityVersion`, `projectName`, and `isPlaying`. Read those back in plain language ("Connected to <projectName> on Unity <unityVersion>, <toolCount> tools available").

If it times out or errors, stop and help fix it first — the three usual causes:
- **The proxy isn't wired into Claude Code.** The MCP server is the `claude-bridge-unity` proxy exe; it must be in your MCP config and Claude Code must have been restarted since (MCP servers load at startup). See the plugin README's install steps.
- **Unity isn't running, or the package isn't installed.** The Unity side is the `com.lousputthole.claudebridge` package (Editor-only). Unity must be open with that package in the project. On launch it writes a status file to `~/.claude-bridge/` that the proxy discovers.
- **Unity is mid-compile / mid-domain-reload.** Give it a few seconds and retry — the proxy buffers across reloads, but a brand-new connection during a reload can need a moment.

You can also confirm liveness with `mcp__claude-bridge-unity__ping` (returns the bridge version + editor time) — it's the lightest possible round-trip.

**3. Get the lay of the land**
- `mcp__claude-bridge-unity__scene_get_info` — the active scene name, path, and root-object count.
- `mcp__claude-bridge-unity__scene_hierarchy` with `maxDepth=1` — a quick top-level look at what's in the scene (cap the depth so you don't dump a huge tree).

Summarize what you see: an empty scene, a default scene (just a Camera + Directional Light), or the user's existing work.

**4. Recommend a first move**
Based on whether the scene is empty and what the user wants, offer 2–3 concrete starts, e.g.:
- "Block out a test scene — a ground plane, a light, and a couple of primitives" (`game_object_create_primitive`).
- "Inspect something — point me at an object and I'll read its components and values" (`game_object_get_info` / `get_component_fields`).
- "Run it — enter play mode and I'll confirm it's running" (`sim_play` → poll `sim_status`).

Keep the first move small and verifiable. The point is a successful round-trip the user can see, not a grand build.

**5. Help + feedback**
- **Troubleshooting:** ask me here anytime — that's what I'm for. If something won't compile or the bridge stalls, I can read the editor state and walk it back.
- **Bugs / feedback:** GitHub issues — https://github.com/LouSputthole/Claude-Bridge-Unity/issues
- Built by **LouSputthole**.

**6. Hand off**
> What do you want to build first?

## Notes
- This is a guide, not a script. Read the room: a returning power user doesn't need the welcome.
- **Scene-mutating tools refuse during play mode** and return a clear error — stop play (`sim_stop`) before editing the scene.
- For anything visual, the discipline that prevents guess-and-check loops lives in the **`claude-bridge-build-feature`** skill (screenshot, read the PNG, verify). Point the user there once they start building.
- To write correct Unity C# (MonoBehaviour lifecycle, Editor vs runtime, AssetDatabase), the **`claude-bridge-api`** skill is the brain. For genre/system recipes, **`claude-bridge-cookbook`** is the router.
