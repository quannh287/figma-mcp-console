---
name: figrelay-generate-design
description: Assemble full screens in Figma out of the file's existing components, styles and variables through the Figma Relay MCP server — instead of drawing rectangles from scratch. Use when the user asks for a new page/screen/flow "using our design system", "with our components", or wants several screens that stay consistent.
---

# Generating screens from a design system

Rule: **instantiate, don't draw.** A screen made of raw rectangles is not a
deliverable when the file has components.

## 1. Inventory first (do not skip)

- `get_local_components` — every COMPONENT / COMPONENT_SET in the file, all pages.
- `get_local_styles` — paint, text, effect styles.
- `get_variable_defs` — collections, modes, tokens.
- `get_metadata` + `list_pages` — where existing screens live; open the right page
  with `set_current_page`.

Screenshot one existing screen (`find_nodes` → `get_screenshot`) to learn the
house layout: page padding, section spacing, header/footer pattern. Copy it.

Report what is missing before building. If there is no Card component, say so and
ask whether to build one or approximate — don't silently fake it.

## 2. Compose

1. Root `create_shape` FRAME at the target size, named like the existing screens,
   placed to the right of them with a clear gap.
2. Resize the root first, **then** `set_auto_layout` VERTICAL on it with the house
   padding and spacing — resizing afterwards resets sizing modes to FIXED.
   Every section inside is itself an auto-layout frame. Nothing absolute.
3. `create_instance` for each component, `set_instance_properties` for its variant
   and text overrides. Swap with `swap_component` rather than detaching.
   **Never `detach_instance`** to "make it fit" — that is how a design system dies.
4. Only `create_text` / `create_shape` for genuinely bespoke content, and then
   style it with `apply_style` / `set_bound_variable`, never a literal hex.

## 3. Verify and iterate

`get_screenshot` per section as you go, not once at the end. Check against the
house screen: alignment, rhythm, and that every colour came from a style or
variable. Fix, re-screenshot, then `set_selection` on the root frame.

## Multiple screens

Build the first screen, screenshot it, **get the user's sign-off**, then clone
the structure for the rest (`clone_node` with `parent_id` and x/y — a clone inside
a SECTION otherwise lands at 0,0). Wire the flow with `set_prototype_links`
between the top-level frames and `set_flow_starting_point` on the entry screen.

## Bulk work

Dozens of identical instances, or rebinding every fill in a subtree, is what
`run_script` is for. Everything smaller stays on the atomic tools — they fail with
a precise error instead of half-writing the document.
