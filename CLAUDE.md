# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A stdio MCP server (Go) that lets AI clients read/write Figma documents in real time through a Figma Desktop **development plugin** — no Figma REST API, no token. Shipped on npm (`figma-mcp-console`) as prebuilt binaries; the plugin is shipped as `plugin.zip` on GitHub Releases.

## Commands

```sh
go test ./...                 # all tests
go test -race ./...           # what CI runs
go test -run TestName ./internal/bridge   # a single test
go vet ./...
go run ./cmd/figma-mcp        # local server, version reports "dev"
go run ./cmd/figma-mcp -port 2001   # or FIGMA_MCP_PORT
./npm/build.sh                # cross-compile 6 binaries + copy plugin into npm/bin/
```

There is no root `package.json` and no Makefile. The npm package lives in `npm/` only.

Contributor setup, the reload/bridge gotchas and the checklist for adding a tool live in `CONTRIBUTING.md`; the reasoning behind the non-obvious tools is in `docs/design-notes.md`.

## Architecture

Three processes, one hop each: **MCP client → Go server (stdio) → WebSocket bridge → Figma plugin (JS sandbox)**.

- `cmd/figma-mcp/main.go` — wires `bridge.Router` + `tools.Register`/`RegisterPrompts`, serves MCP over stdio. stdout is the JSON-RPC transport, so **all logging must go to stderr**. Also self-exits when its parent (the MCP client) dies.
- `internal/bridge` — the shared local bridge. `Router` runs an election loop: the first session to bind port 2000 is the **owner** (serves plugins on `/ws`, other sessions on `/peer`); later sessions become **peers** and forward calls through the owner. When the owner dies, peers re-elect within ~500ms. `bridge.go` is the owner, `peer.go` the client side.
- `internal/tools` — one MCP tool per plugin command. `registerBridged[In]` wires a tool straight to the plugin command **of the same name**, so tool names in `tools.go` and handler keys in `plugin/code.js` must match exactly.
- `plugin/code.js` — main thread; the `handlers` object implements every command against the Figma Plugin API. `plugin/ui.html` — the WebSocket client to the bridge (the main thread has no network) plus the status UI.

### Things that must stay in sync

- **Port 2000** appears in `main.go` (`defaultPort`), `plugin/manifest.json` (`devAllowedDomains`) and `ui.html`.
- **`bridge.ProtocolVersion`** — bump on any breaking change to the command set or message shape; `ui.html` carries the matching constant and warns in its status line on mismatch.
- **Version** — single source of truth is `npm/package.json`; `npm/build.sh` injects it via `-ldflags -X main.version=`. Tagging `vX.Y.Z` triggers the release workflow (plugin.zip only; binaries go out via npm).
- A new tool means: args struct + `registerBridged` call in `tools.go` **and** a handler in `plugin/code.js`.

### Multi-file routing

Every tool's args struct embeds `fileArg`; the router picks the plugin window by file name (case-insensitive substring) or key. Omitted when only one file is connected. `list_files` is answered by the bridge itself, not forwarded.

### Where work happens: server vs plugin

The plugin sandbox has no filesystem and no free network, so anything touching either lives in the Go server: `download_assets` writes files, `import_image` fetches/reads bytes and ships base64, `get_screenshot` decodes base64 into `mcp.ImageContent`. All disk paths go through `resolveProjectPath`, which rejects anything escaping the server's cwd (the project the client launched it in).
