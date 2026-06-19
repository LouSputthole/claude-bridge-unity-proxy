# Claude Bridge for Unity — stdio proxy (free)

The **free, self-contained stdio proxy** for **Claude Bridge for Unity** — the Editor package
on the Unity Asset Store that exposes ~300 AI-callable tools inside your running Unity Editor.

You only need this for the **stdio** transport (e.g. the Claude Code plugin). The Unity
package's built-in **HTTP transport needs no download** — if you're on HTTP, ignore this repo.

## Download

Grab the binary for your OS from the [latest release](../../releases/latest):

| OS | Asset |
|----|-------|
| Windows x64 | `claude-bridge-unity-win-x64.exe` |
| macOS (Apple Silicon) | `claude-bridge-unity-osx-arm64` |
| macOS (Intel) | `claude-bridge-unity-osx-x64` |
| Linux x64 | `claude-bridge-unity-linux-x64` |

Verify downloads against `SHA256SUMS.txt`.

- **macOS / Linux:** `chmod +x` the file.
- **macOS (unsigned):** clear quarantine once — `xattr -d com.apple.quarantine ./claude-bridge-unity-osx-*`.

## Use

Register it with your MCP client (it connects to your running Unity Editor and survives domain reloads):

```json
{ "mcpServers": { "claude-bridge-unity": { "command": "C:/path/to/claude-bridge-unity.exe" } } }
```

Full docs ship inside the Asset Store package (`Documentation/`).

---
The proxy is free and redistributable. The paid product is the Unity **Editor package** (the 300 tools).
