---
name: figrelay-generate-library
description: Build or update a Figma design system library from this codebase through the Figma Relay MCP server — tokens as variables, text/paint/effect styles, then components with variants. Runs in phases and stops for review between them. Use when the user asks to "create our design system in Figma", "sync our tokens to Figma", or "turn our component library into Figma components".
---

# Codebase → Figma library

Long, destructive-by-accident work. **Stop for review after every phase.** Never
run the whole thing in one go.

## Phase 1 — Discovery (read only)

- In the repo: find the **token source file** (theme config, design-tokens JSON,
  Tailwind config, CSS custom properties in source form) and the per-component
  style definitions (`tva({...})`, variant maps, style objects). Grep, don't guess.
- In Figma: `get_variable_defs`, `get_local_styles`, `get_local_components`,
  `list_pages` — what already exists.
- **Check the plan's mode limit** before promising light/dark via modes — see
  Phase 2.

**Never reverse-engineer a compiled stylesheet.** A built CSS bundle is hundreds
of KB of utility output, gets truncated by the read cap, and usually hides the
`:root` block past the cut. The config source has the literal values, and the
component's variant block has the real spec — `xs: 'px-3.5 h-8'` gives you
padding 14 / height 32 and the exact token name `bg-primary-500`. That fidelity
is what makes the library match the app; screenshots and built CSS do not.

Report a plan: tokens found, components found, what already exists in Figma, what
you propose to create vs update, and anything ambiguous. **Wait for approval.**

## Phase 2 — Foundations (variables and styles)

Do this phase in `run_script`, not the typed variable tools — a hundred tokens is
a hundred round-trips otherwise. Read the `run_script` rules in
`figrelay-design` first; the ones that bite hardest here:

- **Modes are a plan limit.** `addMode` on a free/starter plan throws
  *"Limited to 1 modes only"*. Probe it on the first collection before building
  anything around light/dark. Fallback: two sibling collections
  (`tokens · Light`, `tokens · Dark`), components bound to Light, and tell the
  user plainly that mode switching is off the table. Discovering this 300
  variables in means redoing the phase.
- **The script that throws keeps what it made.** The collection created on the
  line before a failed `addMode` survives. Every script here is find-or-create
  keyed by name, and adopts an orphan rather than making a second one:
  `let c = cols.find(x => x.name === target) ?? adopt(cols.find(x => x.name === 'tokens'))`.
- **Slash names** (`primary/500`, `space/md`) — that is what makes the groups in
  the Variables panel. Convert `--color-primary-500` on the way in.
- **Seed bound paints from `valuesByMode`** — cheap insurance. In-file bindings
  render correctly even seeded black (Figma normalises at bind time); the seed
  only matters for bindings that cannot resolve.
- Names mirror the code so a human can trace a token back to its source.
- Updating an existing collection: set the value on the existing variable.
  A duplicate collection is the failure mode here.
- `create_paint_style`, `create_text_style`, `create_effect_style` for what
  designers pick from the UI.

**Stop and show `get_variable_defs` output.**

### Splitting or merging collections

Needed whenever the mode limit forces one collection into two. Every step below
is measured; the sequence as a bulk operation is not, so verify as you go.

1. Create the target collection's variables with the same names, copying values
   from the source via `valuesByMode`.
2. Build a `name → newVariable` map.
3. Scan for consumers with `node.boundVariables?.fills?.some(a => ids.includes(a.id))`
   and rebind each hit. Rebinding preserves the rendered value and needs no
   reseed; renaming a collection or variable never breaks a binding.
4. **Re-scan and assert zero consumers of the old collection before removing it.**
   This is the whole safety net: a node left bound to a removed collection keeps
   its alias and its last colour, looks perfectly fine, and is dead — and
   `getVariableByIdAsync` still returns a live-looking object for it, so a null
   check finds nothing. Test with list membership
   (`getLocalVariablesAsync().some(v => v.id === aliasId)`), never fetchability.
5. Only then `collection.remove()` — immediate, with no undo from the API.

## Phase 3 — File structure

One page per group (Foundations, Components, Patterns) via the user's convention;
`set_current_page` before building. Moving finished components to another page
later is safe — cross-page instance references hold (verified over 54 instances);
`set_current_page` in a script is `setCurrentPageAsync`. A labelled section per component, laid out on
a grid with generous spacing — a library people scroll through, not a pile.

## Phase 4 — Components

Per component, simplest first:

1. Build the base frame, **resize it first, then `set_auto_layout`** — resizing
   after configuring sizing modes silently resets them to FIXED and you get a
   component locked at the resized height with its text spilling out. Bind every
   colour/radius/space to the Phase 2 variables (`set_bound_variable`) or styles
   (`apply_style`).
2. Build each variant as its own frame named `Property=Value`
   (`Variant=Primary`, `Size=Large`), `create_component` each, then
   `combine_as_variants`.
3. `add_component_property` for TEXT / BOOLEAN / INSTANCE_SWAP props, bound to the
   child layer they drive, so the property actually does something.
4. `get_screenshot` the set and compare against the real rendered component.

Batch by component family, pause between families.

## Phase 5 — QA

- Every component: screenshot vs the real thing in the app.
- Every fill/radius/spacing: bound to a variable or style, no stray hex
  (`find_nodes` by fill colour catches leftovers).
- Naming matches the code, variants resolve, instances still work after edits.
- Report coverage: what shipped, what was skipped, what needs a designer.

## Never

- Never `remove_nodes` on existing library components to "rebuild cleanly" —
  it breaks every instance across every file. `rename_nodes` and edit in place.
- Never run a phase the user has not approved.
- Never resize a frame after setting its auto-layout sizing modes.
- Never treat a `run_script` error as "nothing happened" — check the document
  first; the throw is often on the return line, after all the work succeeded.
- Never design around variable modes without confirming the plan allows >1.
- Never scrape compiled CSS for tokens when the config source is in the repo.
- Never remove a variable collection without first asserting it has no
  consumers left — the failure is silent and looks correct.
