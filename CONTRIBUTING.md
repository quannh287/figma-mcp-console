# Contributing

Thanks for helping out. This project is a Go MCP server plus a Figma plugin that talk over a local WebSocket bridge — if you can run Figma Desktop and `go test`, you can work on it.

## Prerequisites

- **Go** 1.26+ (`go version`)
- **Node.js** 18+ — only for the npm wrapper and the release build
- **Figma Desktop** — the web app cannot run development plugins

## Run your changes

```sh
# 1. Build the server
go build -o /tmp/figma-mcp ./cmd/figma-mcp

# 2. Point your MCP client at that binary instead of npx.
#    Claude CLI, from the directory you want it registered in:
claude mcp remove figma-console
claude mcp add figma-console -- /tmp/figma-mcp
```

Then in Figma Desktop: **Plugins → Development → Import plugin from manifest…** → pick `plugin/manifest.json` **from your checkout**, and run **Plugins → Development → Figma MCP Console**. Keep the window open; a green dot means it reached the bridge.

Exported assets are written relative to the server's working directory, which is wherever your client launched it.

### Import from the checkout, not from a copy

Figma loads the plugin from **the folder you imported**, and remembers that path. If you imported the release `plugin.zip` or ran `npx figma-mcp-console install-plugin`, Figma is reading a *copy* (e.g. `~/Desktop/plugin`) and your edits in `plugin/` change nothing — the symptom is `unknown command: <your new tool>`, which looks exactly like a missing feature.

The path Figma is actually using is printed under the plugin's name in **Plugins → Development**. If it is not your checkout, re-import from `plugin/manifest.json`, or copy `code.js`, `ui.html` and `manifest.json` over after every edit.

### Applying a change

| You changed | What to do |
|---|---|
| `plugin/*.js`, `plugin/*.html` | Close the plugin window and run it again. No build step — Figma reads the files as they are, but only when the window opens. There is no hot reload. |
| Go server | `go build` again, then make your MCP client restart the server (restart the client). |

### The old server keeps the bridge

Only one process owns port 2000, and the server you just rebuilt is not it until the previous one exits. Every MCP client session spawns its own server, so a second editor or an older session can be holding the bridge with a stale build. The plugin then reports a protocol mismatch even though you just updated it.

```sh
lsof -nP -iTCP:2000 -sTCP:LISTEN   # which process owns the bridge
ps -o pid,ppid,lstart,command -p <pid>   # and which client started it
```

Kill that process (or quit the client that owns it) and the next server takes over within about a second; plugins reconnect on their own.

The status line names the side that is behind — "Server outdated" means update the server, "Plugin outdated" means re-import the plugin. Bump `bridge.ProtocolVersion` and `PROTOCOL` in `plugin/ui.html` together whenever you add or change commands, so that message stays truthful.

## Architecture in one paragraph

`cmd/figma-mcp` serves MCP over stdio. `internal/bridge` runs a shared WebSocket bridge on port 2000: the first session to bind it is the **owner** and serves every plugin window (`/ws`) and every other session (`/peer`); the rest become peers and forward through it. `internal/tools` registers one MCP tool per plugin command. `plugin/code.js` executes those commands against the Figma Plugin API; `plugin/ui.html` is the WebSocket client, since the plugin main thread has no network access. `CLAUDE.md` has more detail.

**stdout is the MCP transport.** Anything you print there corrupts the protocol — log to stderr.

## Adding a tool

A tool exists in four places. Miss one and it fails at runtime, not at compile time:

1. **`internal/tools/tools.go`** — an args struct embedding `fileArg` (and `nodeTarget` if it mutates existing nodes), plus a `registerBridged[...]` call.
2. **`plugin/code.js`** — a handler in the `handlers` object **with exactly the same name**. That is how dispatch works.
3. **`cmd/figma-mcp/main_test.go`** — bump the expected tool count. It is a deliberate tripwire against adding or dropping a tool by accident.
4. **`internal/bridge.ProtocolVersion`** *and* the `PROTOCOL` constant in `plugin/ui.html` — only when you change the message shape or break an existing command. A hand-installed plugin cannot auto-update, so the mismatch warning is what tells users to re-import.

Conventions worth matching:

- Mutating tools take `node_id` **or** `node_ids[]` via `nodeTarget`, and return `done(params, nodes)` — ids by default, full summaries only when `verbose` is set. A batch of 50 echoed summaries buries the caller's context.
- Anything needing the filesystem or the network belongs in the Go server, not the plugin: the sandbox has neither. `download_assets`, `import_image` and `get_screenshot` show the pattern.
- All disk paths go through `resolveProjectPath`, which rejects anything escaping the working directory.
- `jsonschema:"..."` tags are what the model reads to decide how to call your tool. Say what the parameter does and what a valid value looks like.

## Tests

```sh
go test ./...              # all
go test -race ./...        # what CI runs
go test -run TestEndToEnd ./cmd/figma-mcp
```

`TestEndToEnd` boots the real binary and speaks MCP to it; it does not need Figma, since the point is the not-connected path. `TestEveryBridgedToolHasAPluginHandler` catches a tool registered without its handler.

There is no test harness for `plugin/code.js` — CI never runs it. At minimum `node --check plugin/code.js`, and exercise anything non-trivial in a real Figma file before opening the PR. Say in the PR what you tested by hand.

## Commits and PRs

- [Conventional Commits](https://www.conventionalcommits.org/): `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`, with `!` or a `BREAKING CHANGE:` footer for breaks.
- One topic per PR. Say what changed, why, how you verified it, and how to roll it back.
- CI runs `go vet ./...` and `go test -race ./...` on every PR, and can be run on demand from the Actions tab.

## Releasing (maintainers)

```sh
vim npm/package.json          # version is the single source of truth
./npm/build.sh                # 6 binaries + plugin copy -> npm/bin/
cd npm && npm publish
git tag v0.6.0 && git push origin v0.6.0
```

The tag is what triggers `release.yml`, which packages `plugin/` into `plugin.zip` and creates the GitHub Release. Binaries ship through npm only — no workflow publishes them. Publish and tag together: a release whose plugin speaks a newer protocol than the published server makes every user see the outdated warning.
