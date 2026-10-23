# MCP client setup

Every client runs the same command: `npx -y figrelay@latest`. No install step — `npx` downloads the binary on first run. Below is the per-client detail that does not fit in the README.

## Claude CLI

```sh
claude mcp add figma-relay -- npx -y figrelay@latest              # this project only
claude mcp add --scope user figma-relay -- npx -y figrelay@latest # all projects
```

Both behave the same at runtime: each session starts its own server in that project's directory (so exported assets land in the right project), and all instances share one local bridge. A project-level `.mcp.json` with the canonical block works too.

## Claude Desktop

**Settings → Developer → Edit Config** (`claude_desktop_config.json`), then restart the app. On Windows use `"command": "cmd"` with `"args": ["/c", "npx", "-y", "figrelay@latest"]`.

## Codex CLI

```sh
codex mcp add figma-relay -- npx -y figrelay@latest
```

Or in `~/.codex/config.toml`:

```toml
[mcp_servers.figma-relay]
command = "npx"
args = ["-y", "figrelay@latest"]
```

## VS Code (GitHub Copilot)

The only client needing a different shape — `servers`, not `mcpServers`. Create `.vscode/mcp.json` (or **Command Palette → "MCP: Add Server"**) and use Copilot Chat in **Agent** mode:

```json
{
  "servers": {
    "figma-relay": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "figrelay@latest"],
      "cwd": "${workspaceFolder}"
    }
  }
}
```

## Cursor

`.cursor/mcp.json`, or **Settings → MCP → Add new MCP server**. Canonical block.

## Antigravity

Agent panel → **MCP Servers → Manage MCP servers → View raw config** (`mcp_config.json`). Canonical block, then click **Refresh**.

## Anything else

opencode, Windsurf, Cline, Gemini CLI, Zed and the rest speak standard MCP stdio: command `npx`, args `["-y", "figrelay@latest"]`.

## Multiple files and projects

All sessions and all Figma files share **one local bridge** on port 2000 (override with `-port` or `FIGMA_MCP_PORT`). The first server to start owns it, later ones join, and if the owner exits another takes over within about a second. Nothing to configure.

- **Several Figma files** — run the plugin in each file. With more than one connected, every tool accepts a `file` parameter (file name, case-insensitive substring, or key) and `list_files` shows what is connected, so one session can copy a color from *Design System* into *Landing Page*. With one file connected, nothing changes.
- **Several code projects** — each session reaches every connected file through the shared bridge. Name the file you mean in the prompt.
