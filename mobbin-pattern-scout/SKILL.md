---
name: mobbin-pattern-scout
description: Competitive UI/UX analysis for a screen redesign. Pulls Mobbin references, places them in Figma beside the user's design, compares competitors' information architecture, and turns the findings into wireframes and design-system hi-fi variants.
---

# Mobbin Pattern Scout

A product-design workflow for redesigning one screen (e.g. "My Account") using competitor patterns from Mobbin, with Figma as the working canvas.

The loop: **references → IA analysis → wireframes → designer corrections → hi-fi → targeted re-search when stuck on a decision.**

## Requirements

- Mobbin MCP (`search_screens`, `search_flows`)
- Figma MCP (`create_session`, `get_screenshot`, `get_metadata`, `use_figma`, `upload_assets`). Load the `figma-use` skill before any `use_figma` call, and `figma-generate-design` before building screens.
- A Figma link to the screen being redesigned.

## Ground rules

- **Evidence, not verdicts.** Competitor frequency is input. Product bets (e.g. gamification placement, brand moments) belong to the designer. Before generating layouts, ask which decisions are fixed, or state your assumptions on the canvas.
- **Be honest about coverage.** Mark anything Mobbin didn't capture. Never imply a page is complete when it isn't.
- **Don't overwrite work.** For every iteration, duplicate frames into a new labelled board or section (v2, v3…). Only edit a frame in place when the designer explicitly asks for an edit.
- **Check visually after every write.** Screenshot the result and look for overflow, clipped text, icons outside containers and invisible text before reporting back.
- **Keep the designer's structure.** When they hand you a wireframe (e.g. "B5"), the hi-fi must follow its structure, not the original screen's.

## Workflow

### 1. Understand the target screen
- `create_session` with the file key, then `get_screenshot` and `get_metadata` on the node.
- List its sections top to bottom and what each one does. Note the product type (e.g. cashback/rewards app) to steer the searches.

### 2. Search references (broad pass)
- Run 2–3 `search_screens` queries in parallel on the same platform. Use one intent per query, e.g. "cashback rewards app account screen with earnings balance" and "shopping app my account screen with wallet balance and membership tier".
- Look at the returned images; don't pick from metadata. Choose about 5 that each teach something specific: a direct competitor, a tier/membership pattern, a dense-but-organised hub, a progress-first layout.
- Cite every screen with its `mobbin_url`.

### 3. Get full pages, not just the first fold
- Mobbin iOS screens are single viewports (about 1179×2676). Scrolled parts are stored as **separate screens inside flows**, and only for some apps.
- For each chosen app, run `search_flows` with "<App> account tab". Find the flow that contains the screen you picked and take its scrolled segments.
- Check segments visually: drop modals, edit-profile screens and different-state duplicates. Make a contact sheet (PIL) to compare candidates quickly.
- Where part of a page isn't captured, say so. On the canvas, add a dashed orange box reading "Not captured on Mobbin".

### 4. Place references in Figma
1. Make a board with `use_figma`: a title, then one column per app (app name with a hyperlink to `mobbin_url`, then grey rectangles sized 375×851 for each segment).
2. Download the images: `curl -sfL -o name.jpg "<image_url>?format=jpg"`. Use the `image_url`, not the inline preview.
3. Call `upload_assets` with `nodeIds` set to the rectangles, then POST each file to its `submitUrl` (multipart `file=@...;type=image/jpeg`). The POSTs can run in parallel.
4. Position new boards clear of existing content (to the right of the rightmost node).

### 5. Analyse information architecture
Build an IA board in Figma:
- **Section maps:** one column per app, the target screen first. Number the sections top to bottom and give each a colour bar for its job: Identity & status, Balance & earnings, Owned items & activity, Gamification & progress, Promos & cross-sell, Profile completion, Account & settings, Help & legal, Mixed purpose.
- **Matrix:** for each content type, the section number where it first appears in each app ("—" if absent).
- **Key findings with numbers:** for example, "10 sections vs competitor average 6.6", "money split across 3 blocks", "no settings/help on page", "two entry points to the same history".
- State data limits: gaps in captures and tagging judgement calls.

### 6. Generate wireframe directions
- Make 3 lo-fi grayscale wireframes at 375 width, each a different direction built on the findings, e.g. A "Wallet first" (Rakuten-like), B "Status first" (Marriott/adidas-like), C "Compact hub" (Grab/foodpanda-like).
- Under each, add an annotation card: "What it fixes" (mapped to the findings) and "Trade-off".
- Add a red **first-fold line at 812 pt** with a label, so fold placement can be checked.
- Expect designer corrections such as:
  - must-stay-above-fold items (e.g. Achievements)
  - hierarchy between figures (e.g. lifetime earnings above available-to-withdraw)
  - grouping (e.g. lifetime earnings with tier)
  - removals

  Apply each round as a new labelled board (v2, v3, v4…). Add a "What changed" note and re-check the fold.
- If an item doesn't fit in its slot (a long label in a pill, for instance), change the component (e.g. a segmented link row); don't let it overflow.

### 7. Build the hi-fi with the product's design system
- **Clone the existing production screen** (it already uses the design system's components, text styles and variables) and restructure it into the chosen wireframe. Don't redraw anything.
- Get tokens from the original's bindings: walk `boundVariables` and use `getVariableByIdAsync`. Importing by key can fail. Bind fills, strokes and radii to tokens (border subtle, radius m/xl, background primary/secondary, font/icon primary/inverse, brand colour).
- Change component variants with `setProperties`, e.g. a top bar's `background` default/transparent, a status bar's `Dark Mode`, a button's `variant`, and the icon wrapper's `hasBackground:'false'` after a top bar variant swap.
- Scale badge instances with `rescale()`. Collapse sections with `visible=false`. Flip a chevron with `rotation=180`.
- Use full-bleed brand art (e.g. a header background component) as an **absolutely positioned child** of a clipped card. Lock it so it can't be dragged and break the layout.
- Recompute the frame height after restructuring (`main.y = card.y + card.height`, then `resize`). Re-add the fold line.

### 8. Resolve decisions with a targeted second search
When the designer is torn between two options (e.g. full-bleed brand header vs white header + contained card):
- Search each pattern separately with a specific query, not a topic.
- Build a board with one labelled row per pattern (plus any hybrid that shows up). Include **the product's own live app** if Mobbin has it, since it shows what users already know.
- Give a recommendation backed by evidence (e.g. "none of the full-bleed refs run brand colour past ~180 pt; yours runs ~460 pt"), plus the cost of the other option.
- If asked, build hi-fi variants that each follow one reference app's header pattern, with the same IA and components.

## Known pitfalls

| Problem | Cause | Fix |
|---|---|---|
| Can't edit the product's text | The brand font (e.g. "ShopBack Sans") isn't available to the plugin runtime | You can still clone, move and recolour fills, and change layout sizing. Characters and `textAutoResize` can't change. Keep the existing copy, and create new labels in Inter Bold named "apply <text style>". List the copy changes the designer must make by hand. |
| New text renders with width 0 | Font is listed but has no usable glyphs (e.g. SF Pro) | Use Inter. Check `width > 0` after creating text. |
| Tile icons spill outside their tiles | Tiles reflowed into a new grid; inner frames' sizing changed | Keep the design system's original grid, or rebuild the row so tiles and inner frames have FILL sizing. Check bounds programmatically. |
| Background stretches a card over the page | Art child lost absolute positioning | Insert as an absolute child after auto layout settles, then lock it. |
| White icons with grey circles after a variant swap | The icon wrapper's `hasBackground` reset to true | `setProperties({hasBackground:'false'})` on the top bar's icon instances. |
| A failed `use_figma` call | — | The whole script rolls back. Check the target nodes exist (`getNodeByIdAsync` can return null after user deletions) before positioning relative to them. |

## Output conventions

- Board names: "Mobbin refs — <Screen>", "IA analysis — <Screen>", "Wireframes vN — <change>", "<Screen> — Hi-fi (<option>) vN", and a section for explorations.
- Every board has a short title, a one-line description, and per-frame notes ("What it fixes / Trade-off" or "What changed").
- In chat, cite Mobbin screens as markdown links. Keep replies to what changed and what's left, with the trade-off stated in numbers.
