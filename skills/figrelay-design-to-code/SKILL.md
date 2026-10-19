---
name: figrelay-design-to-code
description: Turn a Figma frame, component or selection into code in this repository — read the design through the Figma Relay MCP server, map it onto existing components and design tokens, export the assets it needs. Use when the user says "implement this design", "build this screen", "code this component", "match the Figma", or pastes/points at a Figma selection.
---

# Figma → code

Goal: code that matches the design **using this codebase's own components and
tokens**. A pixel-perfect one-off built from raw divs and hex codes is a failure.

## 1. Read the design

- `get_selection` when the user says "this" / "the selected frame"; otherwise
  `find_nodes` by layer name. `get_metadata` / `list_pages` if you are lost.
- `get_design_context` with `compact: true` and a sensible `depth` (2–3 first,
  deeper only where needed). Full depth on a whole screen buries the real
  structure in thousands of tokens.
- `get_screenshot` on the frame. Read the picture and the tree together — the
  tree gives names and numbers, the picture gives intent.

## 2. Map before you write

- `get_variable_defs` and `get_local_styles` → the design's tokens. Match them to
  the repo's theme/tokens by **value and name**; write the repo's token, never the
  raw hex. Report any design token that has no counterpart instead of inventing one.
  Read the repo's token **source** (theme config, tokens JSON, Tailwind config) —
  never a built CSS bundle, which is utility output and gets truncated before the
  `:root` block.
- `get_local_components` → named components in the design. Grep this repo for the
  same concept (Button, Card, Modal) and reuse it. Only build new when nothing fits.
- Auto layout maps onto flex: `direction` → `flex-direction`, `item_spacing` →
  `gap`, padding → padding, sizing modes → `fixed` / `hug` (fit-content) /
  `fill` (flex: 1). Absolutely positioned children are the exception, not the rule.
- Fixed pixel widths in the design usually mean "fill the container" in code.
  Ask what the responsive behaviour is rather than hard-coding 390px.

## 3. Export assets

`download_assets`, one item per node, chosen per node after checking its type and
fills:

- **SVG** for pure vector — icons, logos, path illustrations.
- **PNG at `scale: 2`** for anything with a raster IMAGE fill, a photo, or
  blur/shadow effects SVG cannot represent.
- **JPG** only for large photos where size matters.

Paths follow this project's conventions (check `CLAUDE.md` / `AGENTS.md` and
where existing assets live). Without a convention, defaults are
`assets/icons/<name>.svg` and `assets/images/<name>.<ext>`. Everything is written
relative to the directory the MCP server runs in and cannot escape it.

## 4. Verify

Run the app or the component's story, then compare with `get_screenshot` of the
Figma node side by side. Spacing, font size/weight, radius and colour are the
usual misses. State plainly what you could not match and why.

## Don't

- Don't copy absolute x/y into CSS. Rebuild with flex from the auto layout.
- Don't invent token names. Use the repo's, or flag the gap.
- Don't export a whole frame as one PNG to "save time" — export its real assets.
