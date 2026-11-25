---
name: new-component-checker
description: Audit a proposed new Figma component or variant against an existing design system (e.g. Microsoft 365 / Fluent 2 UI Kit), report what breaks the system, and fix it in the file. Use when someone shares a Figma link to a new card/variant/sub-component and asks "does this follow our design system?" or "fix it".
---

# New Component Checker

Two-phase workflow for Figma design-system work:

1. **Learn the baseline**: read the existing components on the page the user points to and record every property, token, size rule and documentation convention.
2. **Check and fix the new component**: compare the user's new design against that baseline, report the problems ranked by severity, and (when asked) fix them in the file the way a design-system maintainer would.

Tools: Figma MCP (`create_session`, `get_metadata`, `get_screenshot`, `use_figma`, `search_design_system`, `get_libraries`). Load the `figma-use` skill before any `use_figma` call, and also load `figma-generate-library` before building components.

---

## Phase 1: Learn the baseline

### 1.1 Open the file
- Parse the URL: `figma.com/design/<fileKey>/...?node-id=<a>-<b>` gives nodeId `<a>:<b>`.
- `create_session(fileKey, description)` once, then pass the `sessionId` on every file call.
- `get_metadata(nodeId)` for structure, plus one `get_screenshot` for a visual overview.

### 1.2 Inventory the components (read-only `use_figma`)
For every `COMPONENT_SET` on the page, collect:
- `componentPropertyDefinitions`: VARIANT options, BOOLEAN, TEXT, INSTANCE_SWAP (default + preferred values). Read these on the **set**, never on a variant.
- `description` (usage rules often live here).
- For each variant: fill/stroke **variable names** (via `boundVariables` → `getVariableByIdAsync`), stroke weight/align, effect style, corner radius, layout mode, padding, gap, size, and which children bind to which property (`componentPropertyReferences`).
- For nested instances: main component + its set name (e.g. `Button (Fluent 2) / Style=Secondary, Size=Medium`).
- For text: font family/style, size, text style name, color variable.

Compact per-variant table script:

```js
const page = await figma.getNodeByIdAsync('<PAGE_ID>');
await figma.setCurrentPageAsync(page);
const cache = {};
const V = async id => id ? (cache[id] ??= (await figma.variables.getVariableByIdAsync(id))?.name ?? id) : null;
const S = async id => (typeof id === 'string' && id) ? (await figma.getStyleByIdAsync(id))?.name ?? '' : '';
const set = await figma.getNodeByIdAsync('<SET_ID>');
const rows = [];
for (const c of set.children) {
  const f = c.fills[0], s = c.strokes[0];
  const slots = c.children.map(ch => `${ch.name}:${ch.visible ? 'on' : 'off'}${ch.componentPropertyReferences?.mainComponent ? '+swap' : ''}`).join(',');
  rows.push([c.name,
    f ? await V(f.boundVariables?.color?.id) : 'none',
    s ? (await V(s.boundVariables?.color?.id)) + '@' + c.strokeWeight : 'none',
    await S(c.effectStyleId), c.cornerRadius, c.layoutMode,
    c.paddingTop + '/' + c.itemSpacing, Math.round(c.width) + 'x' + Math.round(c.height), slots].join(' | '));
}
return { props: set.componentPropertyDefinitions, description: set.description, rows };
```

Also read the **documentation frame** text (all visible TEXT nodes) to learn the doc structure: title, badges, description, resources, "Copy me" variations, usage, "Available at", examples.

### 1.3 Report the baseline to the user
- Property table (name / type / options / default).
- Size rules (padding, gap, radius per size).
- Style × State token map.
- Usage rules from the description.
- **Inconsistencies already in the existing system** (swap properties not bound on some variants, layout that changes per state, odd stroke widths, inconsistent shadows, missing text styles, property names with trailing spaces). The new component must not copy these, and a reviewer will raise them anyway.

End by asking for the node link of the new design and whether it is a variant of an existing set or a new sub-component.

---

## Phase 2: Check the new component

### 2.1 Inspect
- `get_metadata` + `get_screenshot` on the new node.
- Deep read-only `use_figma` dump: type (FRAME vs COMPONENT vs INSTANCE), parent, every fill/stroke with variable name or `[RAW]`, text font + text style (`NO STYLE` if none), instance main components + whether they are `remote`, auto layout and sizing, hidden layers, property references.
- `get_libraries` then `search_design_system` for components the design *should* have used (avatar, avatar group, menu button, etc.). An empty search result is evidence, not a reason to keep searching synonyms.
- Read the variant options of components already used in the file (e.g. `.Avatar (Fluent 2)` sizes/layouts, `Button (Fluent 2)` styles/states) by finding an existing instance and reading its main component's parent set.

### 2.2 Checklist (rank findings: Blockers, then Should fix, then What's fine)

**Blockers**
- [ ] Is it a `COMPONENT` inside the right `COMPONENT_SET`? A detached `FRAME` copy loses every property. Count what was lost.
- [ ] Fonts and text styles match the system (e.g. Segoe UI + `Web/Body 1` / `Web/Caption 1`). Different metrics silently resize neighbors (e.g. buttons 66 → 68px).
- [ ] Every token comes from **this** system. Foreign variable names (e.g. `_components/paper/elevation-0`, `success/main` = Material UI) or detached copies of another kit's components are blockers.
- [ ] Anything described as a "button" is actually interactive: real Button instance or button-token states (Rest / Hover / Pressed / Focused / Disabled).
- [ ] Touch/click target is ≥ 24×24 (WCAG 2.2 AA). Overlapping avatars shrink each target.
- [ ] No third-party logos or brand images baked into a component; use a neutral placeholder + INSTANCE_SWAP.

**Should fix**
- [ ] Auto layout everywhere (no fixed containers with absolute children, no negative-gap hacks with oversized hidden children).
- [ ] Layer names are meaningful (no `Frame 1010108647`).
- [ ] Copy: verb vs noun ("Hand off to", not "Handoff to"), no trailing spaces, secondary text uses the secondary foreground token.
- [ ] Follow-up states designed (in progress / done / failed) for anything that triggers an action.
- [ ] Alignment consistent with sibling patterns.

**What's fine**: say explicitly which tokens and spacing already match, so the user knows what not to touch.

### 2.3 Setup recommendation
- Build a local sub-component (dot prefix = unpublished, e.g. `.Agent handoff`) with:
  - VARIANT axes kept small (State × Count; keep it under 30 combos).
  - TEXT property for the label, BOOLEAN for label visibility, INSTANCE_SWAP per avatar/icon with preferred values.
- Add it to the **existing** parent set behind a BOOLEAN (default off) instead of a new variant axis, so it works in every existing variant without multiplying variants.
- Expose nested instances so the sub-component's props show in the parent's panel.
- Bind only to existing variables and text styles; never detach to restyle.
- Add a "Copy me" example + usage note to the doc page.

---

## Phase 3: Fix (only when the user asks)

Order matters. Run `use_figma` calls sequentially and return all created/mutated IDs from each.

1. **Check fonts first.** `figma.loadFontAsync({family:'Segoe UI', style:'Regular'})` and `listAvailableFontsAsync()`. If the system font is missing you cannot write that text or apply its text styles. Match whatever the base component currently uses and **tell the user**. Also re-read the base component: it may have changed since Phase 1 (e.g. someone replaced missing fonts with Inter, which strips text styles).
2. **Harvest token IDs from existing components** rather than guessing names: e.g. read `Button (Fluent 2)` variants `Style=Subtle, Size=Small, State=*` for background/foreground/focus/disabled variable IDs, padding, radius and height.
3. **Build the sub-component** (`figma.createComponent()` per variant, then `combineAsVariants`, grid-position the variants, resize the set). Then:
   - Bind fills/strokes with `figma.variables.setBoundVariableForPaint`.
   - Use real system instances (`<remoteComponent>.createInstance()`), not drawn shapes.
   - Avatar stack: horizontal auto layout, `itemSpacing: -4`, `itemReverseZIndex: true`, 2px OUTSIDE ring bound to the surface token, `cornerRadius: 100`.
   - Focus variant: `strokesIncludedInLayout = false` so the outline doesn't shift content.
   - Add properties with `set.addComponentProperty(...)` and bind via `componentPropertyReferences` (`characters`, `visible`, `mainComponent`).
   - Default swaps to a **neutral** placeholder (icon avatar), not a photo or logo.
   - Write a `description` with token mapping, usage rules and keywords.
4. **Wire into the parent set:**
   - `addComponentProperty('<Name>', 'BOOLEAN', false)`.
   - In each variant, wrap the target region in a new auto-layout row inserted at the same index; move any existing `visible` reference from the old region to the new row.
   - Append the sub-component instance (HUG), bind `visible` to the new boolean, and set `isExposedInstance = true`.
   - Verify all original properties still exist and variant heights are unchanged with the boolean off.
5. **Rebuild the user's example** as an instance of the parent with the new boolean on. Carry the user's custom images over as **instance overrides** (copy IMAGE fills onto the avatar image layer). Do not delete the user's original: rename it `[Old – detached, safe to delete] …` and move it aside.
6. **Verify:** one screenshot of the sub-component set, one of the example instance, and a property/binding audit. Fix visual defects with a targeted script, re-screenshot once, then stop.

---

## Final report format
- **Done in Figma:** bullet per created/changed object with key specs (variant count, properties, tokens).
- **Still open:** anything blocked (missing fonts), product decisions (post-action states), and skipped work (doc page if it has no auto layout), each with what the user needs to do or decide.
- Keep it short; the user can open the file for detail.

## Gotchas learned
- Never read `componentPropertyDefinitions` from a variant; use the set.
- Page context resets every `use_figma` call; call `setCurrentPageAsync` once per script.
- The INSTANCE_SWAP default can be a remote component ID that already exists in the file.
- Fluent kit components in the file are often `remote` copies; you can instance them but not edit them.
- A library search that only returns an unrelated kit means the system's components live in the file as remote copies, so reuse those instances.
