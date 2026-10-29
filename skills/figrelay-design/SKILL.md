---
name: figrelay-design
description: Create or modify content on a Figma Design canvas through the Figma Relay MCP server — frames, text, auto layout, components, variants, styles, variables and prototype links. Use whenever the user asks to draw, build, edit, restyle, rename, rearrange or wire anything in a Figma file. Triggers on "in Figma", "add a frame/button/screen", "change this component", "set up tokens/variables", "wire the prototype".
---

# Writing to a Figma file

The plugin edits **the page that is open in Figma**, which the user can change at
any moment. Never assume where your work lands.

## 1. Orient before writing

- `list_files` — only when several Figma files are connected; pass `file` to every
  later call.
- `list_pages` then `set_current_page` — put the right page in front before creating.
- `get_metadata` for the page's top-level layers, `get_selection` for what the user
  is pointing at. In a large file use `find_nodes` (by name, text, type, fill, font)
  instead of walking the tree with `get_design_context`.
- `get_design_context` with `compact: true`. The full form costs several thousand
  tokens for one frame; only drop `compact` when you need raw paint objects.

## 2. Build top-down

1. A root `create_shape` FRAME at an empty spot, sized for the target
   (390x844 mobile, 1440x900 desktop). Everything else goes inside it via `parent_id`.
2. `set_auto_layout` on every container — direction, `item_spacing`, padding,
   sizing modes. Only position with x/y when auto layout genuinely does not fit:
   absolute children do not reflow when content changes.
3. `create_text` for copy, `create_shape` / `create_line` for structure.
   Name every layer the way a designer would ("Header", "CTA / Primary"), never
   "Rectangle 12".
4. Style: `set_fills`, `set_strokes`, `set_corner_radius`, `set_effects`,
   `set_text_properties`.

Reuse before you create: `get_local_components` + `create_instance`,
`get_local_styles` + `apply_style`, `get_variable_defs` + `set_bound_variable`.
A hard-coded hex where a token exists is a bug.

## 3. Verify visually

`get_screenshot` on the root frame after each meaningful chunk — not once at the
end. Compare against the brief, fix with `move_nodes`, `resize_nodes`,
`set_characters`, `set_fills`, `remove_nodes`, screenshot again. A tall node comes
back scaled with a warning; screenshot a child instead of reading a thumbnail.

Finish with `set_selection` on the root frame so the user sees the result.

## 4. Components, variables, prototypes

- **Component**: build the frame, then `create_component`. For variants, name each
  component `Property=Value` (`Size=Large`) *before* `combine_as_variants`.
  `add_component_property` returns the key `set_instance_properties` needs.
- **Variables**: `create_variable_collection` → `add_variable_mode` (light/dark) →
  `create_variable` → `set_variable_value` per mode → `set_bound_variable` on nodes.
  These are for single edits; a whole token set goes through `run_script`
  (see below). **Check the mode limit before designing around modes**: on a
  free/starter plan `add_variable_mode` fails with *"Limited to 1 modes only"* and
  the light/dark-via-modes pattern is simply unavailable — the fallback is two
  sibling collections (`tokens · Light`, `tokens · Dark`) and no mode switching.
  Name variables with slashes (`primary/500`, not `color-primary-500`) — that is
  what groups them in the Variables panel; flat names are unusable past ~50 tokens.
- **Prototype**: `set_prototype_links` only joins **top-level frames on the same
  page**; a nested frame is rejected. Each call replaces that node's reactions, so
  pass all of a node's links together. Check with `get_prototype_links`,
  mark the entry with `set_flow_starting_point`. In `run_script`,
  `setReactionsAsync` has no `WHILE_PRESSING` trigger — use `ON_PRESS`.
  Cloning a frame also copies its flow starting point; `clone_node` warns, scripts
  must remove the duplicate themselves.
- **Overlays can't be bottom-anchored**: `overlayPositionType` is read-only and
  `overlayRelativePosition` needs `MANUAL`. Make the overlay a full-screen frame
  (device size) with a scrim and the sheet placed at the bottom inside it.

## Gotchas that cost time

- **Fonts**: `create_text` fails if the family is not installed locally.
  `list_available_fonts` first when using anything but the file's existing fonts.
- **Rename, don't rebuild**: `rename_nodes`. Deleting and recreating a component
  breaks every existing instance.
- **`resize_nodes` resets auto-layout sizing modes to FIXED.** Resize *before*
  `set_auto_layout`, or re-apply the sizing modes afterwards — otherwise a frame
  set to hug ends up locked at the resized height with its content spilling out.
  It also does not move children unless they have constraints or auto layout.
- **Auto layout doesn't always shrink**: after removing a tall child the parent can
  keep its old height; set `layoutSizingVertical = "HUG"` again on it.
- **Instances**: children have derived ids `I<instance-id>;<child-id>` — accepted
  by every tool, returned by `find_nodes`. Also:
  - `swap_component` invalidates those ids — `find_nodes` again after a swap.
  - Position/size of a child inside an instance can't be overridden
    (`relative-transform` error) — change the main component or use auto layout.
  - Editing a node right after `detach_instance` in the same script can throw
    `object is not extensible`; re-fetch it with `getNodeByIdAsync` first.
  - Overrides survive edits to the main component, so a changed main property may
    not show on instances that overrode it.
- **Strokes**: an `OUTSIDE` stroke on an ELLIPSE with an IMAGE fill renders a
  jagged ring — wrap it in a frame with an `INSIDE` stroke instead. `dashPattern`
  restarts on every edge of a rounded rect, so a progress ring needs an open
  vector path, not dashes.
- **Batch**: most edit tools take `node_ids`; feed `find_nodes` output straight in.

## `run_script` — the bulk path

Default for anything token-shaped (importing 150 variables as 150 calls is absurd)
and for bulk rebinding. Single edits stay on the atomic tools: they fail with a
precise error, while a half-failed script leaves the document half-changed.

- **A thrown script keeps everything it already created.** Write find-or-create
  scripts keyed by name so a retry adopts the orphan instead of duplicating it.
- **Read the line number before retrying.** The throw is often on the *return*
  line — e.g. `return { n: figma.getLocalTextStyles().length }` — meaning the work
  completed and only the summary failed. Inspect the document, then decide.
- **dynamic-page access**: sync accessors throw. Async-only —
  `getLocalVariableCollectionsAsync`, `getVariableByIdAsync`,
  `getLocalTextStylesAsync`, `getLocalEffectStylesAsync`, `getNodeByIdAsync`,
  `setCurrentPageAsync`, `loadAllPagesAsync`, `node.getMainComponentAsync`,
  `setTextStyleIdAsync`, `setEffectStyleIdAsync`, `setReactionsAsync`.
  Still sync: `variables.createVariable`, `createVariableCollection`,
  `setValueForMode`, `valuesByMode`, `createTextStyle`, `createEffectStyle`,
  `combineAsVariants`.
- **Never call an async-only getter inside a `findAll`/`filter` predicate** — it
  crashes the whole traversal (*"findAll callback crashed"*). Collect the nodes
  first, then `for...of` with `await`.
- **Seed a bound paint from the variable's own value** — free insurance, not a
  cure. Figma normalises the stored colour to the variable's resolved value at
  bind time, so an in-file binding seeded black still renders correctly
  (measured). The seed only ever matters where the variable *cannot* resolve —
  a cross-file or library binding — which is untested. Do it anyway; it costs
  one property read.

```js
const col  = (await figma.variables.getLocalVariableCollectionsAsync()).find(c => c.name === 'tokens · Light');
const mode = col.modes[0].modeId;
const v = await figma.variables.getVariableByIdAsync(id);
const paint = { type: 'SOLID', color: v.valuesByMode[mode], opacity: 1 };
node.fills = [figma.variables.setBoundVariableForPaint(paint, 'color', v)];
```

### Rebinding and deleting variables

Measured, not guessed:

- **Rebinding is safe and needs no reseed.** `setBoundVariableForPaint(paint, 'color', newVar)`
  with the stale colour lands on the new variable's value. Mutating
  `paint.boundVariables.color = {type:'VARIABLE_ALIAS', id}` directly works too.
- **Renaming never breaks a binding.** Bindings are by id; a renamed collection
  or variable keeps every consumer. Rename freely.
- **Deleting a collection leaves silent zombies.** Nodes still bound keep the
  alias and the last literal colour — they look right and are dead. Worse,
  `getVariableByIdAsync(deadId)` and `getVariableCollectionByIdAsync` **still
  return live-looking objects with correct names**, so a null check detects
  nothing. The only reliable test is list membership:
  `!(await figma.variables.getLocalVariablesAsync()).some(v => v.id === aliasId)`.
- **There is no consumer API.** `variable.consumers` is `undefined`, and
  `Object.keys()` on a variable or collection returns `["id"]` — they are
  proxies, so don't introspect them. To find consumers, scan the tree via
  `node.boundVariables?.fills?.some(a => ids.includes(a.id))` (cheaper than
  digging through each paint's `boundVariables`). A plain property read like
  this is fine inside a `findAll` predicate; an async getter is not.
- **`collection.remove()` is immediate with no API undo.** Re-scan and assert
  zero remaining consumers *before* removing — you get no error if you miss some.

- Split by **logical group**, not node count — the point is error isolation.
  Scripts creating 300+ variables, or several hundred nodes, run without timing out.
- Pass a `token` anyway (it costs nothing). A timed-out script **keeps running and
  keeps writing**, so the token is what lets a retry join the original run instead
  of duplicating it. Never redo a timed-out script's work blind.

Raw-API-only gotchas (the atomic tools already handle these):
`figma.combineAsVariants` neither names nor lays out the set — set `name`,
`layoutMode`, `layoutWrap`, `primaryAxisSizingMode = 'FIXED'`, `resize`, padding
and spacing yourself, or every variant stacks at 0,0. `layoutAlign = 'STRETCH'`
does not stretch a child unless `primaryAxisSizingMode` is also `FIXED`.
`layoutPositioning = 'ABSOLUTE'` must be set *after* `appendChild`.
