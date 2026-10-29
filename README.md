# Figma Relay

[![npm](https://img.shields.io/npm/v/figrelay?color=cb3837&logo=npm)](https://www.npmjs.com/package/figrelay)
[![MCP](https://img.shields.io/badge/MCP-Compatible-blue)](https://modelcontextprotocol.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://github.com/quannh287/figma-relay/blob/main/LICENSE)

**Let your AI assistant read and edit your Figma files directly.** No API token, no rate limits, works with any MCP client (Claude, Cursor, Copilot, Codex…).

Use it to:

- **Design → code**: implement a Figma frame in your project, with its icons and images exported.
- **Code → design**: draw screens from a requirement, build a component library from your codebase.
- **Bulk edits**: rename layers, rebind colors to variables, restyle text across a whole page.

## How it works

```text
AI client ──stdio──▶ figrelay server ──localhost:2000──▶ Figma Relay plugin ──▶ your Figma file
```

The server runs on your machine and talks to a plugin window open inside **Figma Desktop**. The plugin does the actual reading and writing through Figma's Plugin API — that is why no token is needed, and why **the plugin window must stay open** while the AI works.

## Setup

You need [Node.js](https://nodejs.org) 18+ and the **Figma Desktop** app (the browser version can't run development plugins).

### 1. Add the server to your AI client

For Claude Code:

```sh
claude mcp add figma-relay -- npx -y figrelay@latest
```

Most other clients take this JSON in their MCP config file:

```json
{
  "mcpServers": {
    "figma-relay": {
      "command": "npx",
      "args": ["-y", "figrelay@latest"]
    }
  }
}
```

Config file locations for Claude Desktop, Codex, VS Code, Cursor and Antigravity are in [docs/mcp-client-setup.md](docs/mcp-client-setup.md). VS Code uses a slightly different shape.

### 2. Install the plugin (once)

1. Run `npx figrelay@latest install-plugin`. It saves the plugin to a folder and opens it. Or download [plugin.zip](https://github.com/quannh287/figma-relay/releases/latest) and unzip it.
2. In Figma Desktop: **Plugins → Development → Import plugin from manifest…** and pick `manifest.json` from that folder.

### 3. Run the plugin

Open your design file and run **Plugins → Development → Figma Relay**. A 🟢 green dot means it is connected to your AI client. Keep the window open; you can collapse it to a dot.

Working in several files? Run the plugin in each one.

### 4. Try it

In Figma, right-click a frame → **Copy link to selection**, then ask your AI:

```text
Implement this Figma frame in our project: <paste link>.
Export its icons, then screenshot the frame and compare with your result.
```

Or the other way round:

```text
Design a pricing page in Figma (1440px wide) using auto layout,
then screenshot it and fix anything that looks off.
```

Ending with "screenshot and compare" makes the AI check its own work.

## What the AI can do

53 tools, grouped by job:

| Job | Tools |
|---|---|
| Look around | `list_files`, `list_pages`, `get_metadata`, `get_selection`, `get_design_context`, `find_nodes`, `get_screenshot` |
| Create | `create_shape`, `create_text`, `create_line`, `import_image`, `clone_node` |
| Edit | `set_fills`, `set_strokes`, `set_effects`, `set_corner_radius`, `set_auto_layout`, `set_text_properties`, `set_characters`, `move_nodes`, `resize_nodes`, `rename_nodes`, `group_nodes`, `remove_nodes`… |
| Components | `create_component`, `create_instance`, `combine_as_variants`, `add_component_property`, `swap_component`… |
| Styles & variables | `create_paint_style`, `apply_style`, `create_variable`, `set_bound_variable`… |
| Prototype | `set_prototype_links`, `get_prototype_links`, `set_flow_starting_point` |
| Export | `download_assets` (SVG / PNG / JPG into your project) |
| Bulk | `run_script` — one JavaScript call instead of hundreds of small ones |

## Agent skills (recommended)

Tools say *what* the AI can call; skills teach it *how* to do the job well: check the file before writing, use auto layout, reuse existing components and tokens, verify with a screenshot.

```sh
npx figrelay@latest install-skills   # writes .claude/skills/figrelay-*/SKILL.md
```

| Skill | Used for |
|---|---|
| `figrelay-design` | Any edit on the canvas |
| `figrelay-design-to-code` | "Implement this design" |
| `figrelay-generate-design` | New screens from the file's existing components |
| `figrelay-generate-library` | A Figma design system built from your codebase |

Other clients can load the same files from [`skills/`](skills/). Edit them to match your conventions.

## Configuration

**Hide tools you don't use.** Every tool takes up context in every session. Set `FIGMA_MCP_DISABLE` to hide whole groups — `components`, `styles`, `variables`, `prototype`:

```json
"figma-relay": {
  "command": "npx",
  "args": ["-y", "figrelay@latest"],
  "env": { "FIGMA_MCP_DISABLE": "variables,prototype" }
}
```

**Tell the AI where assets go.** Exported files are written relative to the folder the server was started in. Add your rules to the project's AI rules file (`CLAUDE.md`, `AGENTS.md`, `.cursorrules`…):

```markdown
## Figma assets
- Icons: SVG into src/assets/icons/, kebab-case names.
- Images: PNG at 2x into src/assets/images/.
```

**Port.** Everything shares one local port, 2000. Change it with `FIGMA_MCP_PORT` (the plugin also expects 2000, so only change it if you rebuild the plugin).

## Troubleshooting

| You see | Do this |
|---|---|
| "no Figma plugin is connected" | Run **Plugins → Development → Figma Relay** in the file, and keep its window open |
| Plugin stuck on "Searching for MCP server…" | Your AI client hasn't started the server yet. Send it a message, or restart the client |
| "plugin disconnected before answering" | The plugin window was closed or reloaded. Run it again and check the file before retrying |
| "N Figma files are connected" | Tell the AI which file to use, or close the extra plugin windows |
| Server doesn't show up in the AI client | Check `node -v` is 18 or newer, then restart the client |
| Exported files in the wrong folder | Register the server in the project's config, or give the AI a full path |
| Font error when creating text | That font isn't installed on your machine; the error names it |

## More

- [docs/mcp-client-setup.md](docs/mcp-client-setup.md) — per-client setup, several files or projects at once
- [docs/design-notes.md](docs/design-notes.md) — why the non-obvious tools work the way they do
- [CONTRIBUTING.md](CONTRIBUTING.md) — build from source, add a tool

**Coming from figma-mcp-console?** Change the package to `figrelay@latest`, re-import the plugin, and use `create_shape` instead of `create_frame` / `create_rectangle` / `create_ellipse`.

Fork of [figma-mcp-console](https://github.com/hoangann2000/figma-mcp-console) by [Hoang Le Thien An](https://github.com/hoangann2000), maintained by [@quannh287](https://github.com/quannh287). [MIT](https://github.com/quannh287/figma-relay/blob/main/LICENSE). Independent community project, not affiliated with Figma, Inc.
