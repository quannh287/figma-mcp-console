# Figma Relay

[![npm](https://img.shields.io/npm/v/figrelay?color=cb3837&logo=npm)](https://www.npmjs.com/package/figrelay)
[![MCP](https://img.shields.io/badge/MCP-Compatible-blue)](https://modelcontextprotocol.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://github.com/quannh287/figma-relay/blob/main/LICENSE)

> **Let your AI assistant read and write Figma — no API token, no rate limits.**
> An MCP server that talks to a Figma Desktop plugin, so the AI can implement a frame in your codebase, draw a screen from a requirement, sync design tokens or bulk-edit layers. Works with any MCP client.

Fork of [figma-mcp-console](https://github.com/hoangann2000/figma-mcp-console), published on npm as [`figrelay`](https://www.npmjs.com/package/figrelay); the plugin shows as **Figma Relay**. **Migrating?** Swap the package name to `figrelay@latest`, re-import the plugin from this release, and replace `create_frame`/`create_rectangle`/`create_ellipse` with `create_shape`.

## ⚡ Quick start

Needs [Node.js](https://nodejs.org) 18+ and the **Figma Desktop** app (the web app cannot run development plugins).

**1. Add the server to your AI tool.** Most clients take this block; [docs/mcp-client-setup.md](docs/mcp-client-setup.md) has the config-file path and the one-line CLI command for each.

```json
{
  "mcpServers": {
    "figma-console": {
      "command": "npx",
      "args": ["-y", "figrelay@latest"]
    }
  }
}
```

| Client | Where the config lives |
|---|---|
| Claude CLI | `claude mcp add figma-console -- npx -y figrelay@latest`, or `.mcp.json` |
| Claude Desktop | Settings → Developer → Edit Config (`claude_desktop_config.json`) |
| Codex CLI | `codex mcp add figma-console -- npx -y figrelay@latest`, or `~/.codex/config.toml` (TOML) |
| VS Code (Copilot) | `.vscode/mcp.json` — **different shape**, see the doc |
| Cursor | `.cursor/mcp.json` |
| Antigravity | MCP Servers → Manage → View raw config (`mcp_config.json`) |
| Others | Standard MCP stdio: command `npx`, args `["-y", "figrelay@latest"]` |

**2. Install the Figma plugin.** Download [plugin.zip](https://github.com/quannh287/figma-relay/releases/latest) and unzip it, or run `npx figrelay@latest install-plugin` (saves it to a folder and opens it). Then in Figma Desktop: **Plugins → Development → Import plugin from manifest…** → pick `manifest.json` (one-time), and run **Plugins → Development → Figma Relay**. Keep the plugin window open — 🟢 green dot means connected. Working across several Figma files? Run the plugin in each one.

**3. Try it.** Right-click a frame → **Copy link to selection**, then ask:

```text
Implement this Figma page in our project: <paste link>. Match layout, spacing,
type and colors; export its icons/images into the project; screenshot the frame
and compare when done.
```

Or the other direction: *"Design a SaaS landing page in Figma (desktop, 1440px): hero + CTA, three feature cards, pricing, footer. Light theme, auto-layout, then screenshot and fix anything off."* Asking for a **screenshot and compare** at the end makes the AI check its own work.

## 🎓 Agent skills

The tools tell the AI *what it can call*; the skills tell it *how to do the job well* — orient before writing, auto-layout over absolute positioning, reuse components and tokens, screenshot and verify. `npx figrelay@latest install-skills` writes them to `.claude/skills/<name>/SKILL.md`; other clients take the same files in their own skills folder. Sources live in [`skills/`](skills/) — edit them to match your conventions.

| Skill | When it fires |
|---|---|
| `figrelay-design` | Anything that writes to a canvas — frames, text, components, variants, variables, prototype links |
| `figrelay-design-to-code` | "Implement this design" — map a frame onto the repo's components and tokens, export assets |
| `figrelay-generate-design` | New screens built from the file's existing components and styles |
| `figrelay-generate-library` | Build or update a Figma design system library from the codebase, in reviewed phases |

## ✂️ Hide tools you don't use

Every listed tool costs context, even in sessions that never touch Figma. `FIGMA_MCP_DISABLE` (or the `-disable` flag) hides whole groups: `components`, `styles`, `variables`, `prototype`. Hiding all four cuts the tool list by about a third.

```json
"figma-console": {
  "command": "npx",
  "args": ["-y", "figrelay@latest"],
  "env": { "FIGMA_MCP_DISABLE": "variables,prototype" }
}
```

## 🧠 Teach your AI what to export, and where

Exports follow the extension the AI picks (`.svg` for vector, `.png`/`.jpg` for bitmap, optional 0.5–4x scale), so put your preferences in the project rules file once — `CLAUDE.md`, `AGENTS.md` (Codex/opencode), `.cursorrules`, `.github/copilot-instructions.md` or `.windsurfrules`:

```markdown
## Figma asset rules
- Icons: SVG into public/icons/. Images: PNG into public/images/.
- File names: kebab-case English (arrow-left.svg, hero-banner.png).
- Read design tokens with get_variable_defs; map them to our theme config.
- After building a screen, screenshot and fix issues before finishing.
```

## 🔧 Troubleshooting

| Problem | Fix |
|---|---|
| ❌ "no Figma plugin is connected" | Open the plugin (**Plugins → Development → Figma Relay**) and keep its window open — in every file the AI should touch |
| 🕐 Plugin says "Searching for MCP server…" | Normal until your AI client connects. If it never does, click the status line to retry, or restart the client |
| 🗂 "N Figma files are connected; pass the file parameter" | More than one plugin window is open — name the file you mean, or close the extras |
| 🚫 The server doesn't appear in your AI tool | Check Node.js 18+ (`node -v`) |
| 📁 Exported files land in the wrong folder | Exports are relative to where the server runs. Register it in the project-level config, or give the AI an explicit path |
| 🔤 Font errors when creating text | The file uses a font not installed on your machine — the error names it |

## 🛠 More

Several Figma files or projects at once: [docs/mcp-client-setup.md](docs/mcp-client-setup.md). Setup, adding a tool and the gotchas that cost time: [CONTRIBUTING.md](CONTRIBUTING.md). Why the non-obvious parts work that way: [docs/design-notes.md](docs/design-notes.md). License: [MIT](https://github.com/quannh287/figma-relay/blob/main/LICENSE) — © [Hoang Le Thien An](https://github.com/hoangann2000) (original project), fork maintained by [@quannh287](https://github.com/quannh287). Independent community project, not affiliated with or endorsed by Figma, Inc.; "Figma" is a trademark of Figma, Inc.
