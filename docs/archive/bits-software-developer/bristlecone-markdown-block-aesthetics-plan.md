# Bristlecone Markdown block — aesthetic refresh plan

**Status:** PLAN ONLY — no code, no PR, no plugin-repo edits  
**Prepared for:** Mahler via BITS (WordPress Plugin Developer)  
**Date:** 2026-09-20 (ET)  
**Repo under plan:** `Zyniker13/bits-markdown` (WP.org / plugin: Bristlecone Markdown)  
**Current version:** **1.1.1** (`bristlecone-markdown.php`, `readme.txt` Stable tag)  
**Target version when implemented:** **1.2.0** (minor / **b** bump — aesthetic feature, not a patch)

**References**
- Local / GitHub sources reviewed: `assets/js/block.js`, `assets/css/editor.css`, `blocks/markdown/block.json`, `assets/css/frontend.css`, `includes/Assets.php`
- Jetpack: `Automattic/jetpack` → `projects/plugins/jetpack/extensions/blocks/markdown/` (`edit.jsx`, `editor.scss`, `controls.jsx`)
- Screenshots (parent agent attachments, 2026-09-20 ~4:32 PM ET): current boxed Bristlecone empty state vs Jetpack empty “Write your _Markdown_ **here**…” state
- Impeccable process (FlexInvoice Developer confirmed): `Zyniker13/FlexInvoice` (`DESIGN.md`, `PRODUCT.md`, `docs/impeccable.md`, `.cursor/skills/impeccable/`, `.cursor/skills/flexinvoice-ui/SKILL.md`); account note `Zyniker13/Documentation` → `cursor-and-bots/08-impeccable-design-skills.md`
- Live FlexInvoice (process demo only): https://flexinvoice.us

---

## 1. Current vs target visual language

### Current (Bristlecone / BITS Markdown block)

From the **current** editor screenshot + code:

| Aspect | Behavior today |
| --- | --- |
| Source UI | `wp.components.TextareaControl` with class `bristlecone-markdown-source`, `rows: 16`, label “Markdown” (visually hidden) |
| Chrome | Default Component Library textarea: **visible border**, white fill, **browser resize handle** — reads as a form field under “Add title”, not as document body |
| Empty state | No inviting placeholder copy; empty boxed field |
| Editor CSS | `editor.css` forces **monospace** (`ui-monospace…`) at 13px / 1.5 / min-height 280px on the textarea |
| Preview | `.bristlecone-markdown-preview` keeps a **1px `#dcdcde` border**, 12×16 padding, min-height 280px; also carries front-end class `.bristlecone-markdown` for content styling |
| Toolbar | `BlockControls`: Markdown / Preview icon buttons + Insert image |
| Inspector | `PanelBody` “Bristlecone Markdown” with help prose |
| Stack | Plain `wp.*` IIFE in `assets/js/block.js` — **no build step** |

### Target (Jetpack-like, typography-first)

From the **preferred** Jetpack empty-state screenshot + Jetpack source:

| Aspect | Jetpack pattern to emulate (feel, not brand) |
| --- | --- |
| Empty + unselected | Seamless `<p class="…__placeholder">` copy: `Write your _Markdown_ **here**…` — **no box**, opacity ~0.62, inherits editor/theme type (serif in Mahler’s theme screenshot) |
| Selected source | `PlainText` with **transparent** focus (no border/box-shadow); monospace code-editor face for typing Markdown |
| Unselected with content | Shows **preview** (rendered Markdown), not the source field |
| Toolbar | Text “Markdown” / “Preview” tab buttons in the block toolbar (pressed state) |
| Overall | Typography and whitespace carry the UI; chrome is almost invisible until selected |

### Gap summary (what the refresh must close)

1. **Remove the boxed TextareaControl look** (border + resize grip + form-control density).
2. **Add an empty / unselected placeholder** that feels like writing, not configuration.
3. **Keep Source + Preview** (Bristlecone’s strength: server preview) while making both states quieter.
4. **Do not** ship Jetpack logos, Jetpack purple, or Jetpack block naming — only the writing-surface language.

---

## 2. UX states to design

Keep capability; redesign presentation.

| State | Intent | Proposed treatment |
| --- | --- | --- |
| **Empty + unselected** | Invite writing; match post canvas | Placeholder paragraph (Jetpack-style copy or Bristlecone-owned variant — open question). No border, no min-height box. Soft opacity. Inherit theme/editor body type for this state (not forced FlexInvoice Newsreader). |
| **Selected + Source** | Edit Markdown comfortably | `PlainText` (or equivalent borderless multiline). Monospace for source fidelity. Transparent focus ring or Gutenberg-standard focus only. Auto-grow / min-height via CSS, **no** native resize handle. |
| **Selected + Preview** | Trust server render | Preview surface without heavy admin card border; still use `.bristlecone-markdown` content rules so preview ≈ front-end. Optional subtle empty preview (“Nothing to preview yet”). |
| **Unselected + has content** | Read like the post | Prefer **preview** when not selected (Jetpack does this). Avoid leaving a huge empty monospace box in the canvas. |
| **Toolbar** | Mode switch + insert image | Keep Markdown / Preview / Insert image. Prefer clearer pressed tab styling (Jetpack uses text tab buttons; Bristlecone can keep Dashicons **or** move to labeled tabs — open question). Do not add Jetpack-only controls. |
| **Inspector** | Orientation, not chrome | Keep PanelBody help; shorten/clarify if needed. No new visual “settings skin.” Compatibility alias blocks keep their existing notice. |
| **Error** | Preview REST failure | Keep `Notice`; ensure it doesn’t reintroduce a heavy box around the whole block. |
| **Math in preview** | Unchanged | Existing KaTeX `renderMath` on preview tab / preview mount. |

**Must preserve:** Source ↔ Preview switching, image insert → Markdown snippet, REST preview debounce, alias/compatibility edit path, `html` attribute persistence for own block.

---

## 3. PlainText vs TextareaControl; placeholder; fonts

### Recommendation: **PlainText** (Jetpack-like), stay on plain `wp.*` JS

- `wp.blockEditor.PlainText` is available without a bundler (same stack as today).
- Drop `TextareaControl` for the source surface so Component Library field chrome goes away.
- No JSX/build required for this swap.

**Optional Jetpack behaviors to consider (not all required in v1):**

- Empty + Backspace removes the block (`onKeyDown` + `removeBlocks`) — nice parity; confirm with Mahler (open question).
- Auto-focus PlainText when selected on Source tab.
- When deselected while on Preview, snap back to Source for next select (Jetpack) **or** remember last tab (current Bristlecone) — prefer **remember last tab** unless Mahler wants Jetpack parity.

### Placeholder strategy

Jetpack does **not** put the famous string on the `<textarea>` `placeholder` attribute. It renders a **static paragraph** when `!isSelected && isEmpty`. Recommended:

1. Empty unselected → placeholder element with i18n string.
2. Selected empty Source → either blank PlainText or a light `placeholder` prop if PlainText supports it cleanly; prefer matching Jetpack (blank field once focused).
3. Copy candidates:
   - Parity: `Write your _Markdown_ **here**…` (Jetpack string; underscores/asterisks literal — teaching Markdown).
   - Bristlecone-voiced: e.g. `Write Markdown here…` or iA Writer–tinged line — **Mahler chooses** (open question).

### Theme-font inheritance vs forced serif

| Surface | Font policy |
| --- | --- |
| Empty placeholder | **Inherit** editor/theme (so it sits under “Add title” like Jetpack in Mahler’s screenshot). Do **not** hardcode Newsreader / FlexInvoice paper stack. |
| Source editing | Keep **monospace** (current `editor.css` + Jetpack `__editor` pattern) for Markdown readability. |
| Preview / front-end | Theme + existing `.bristlecone-markdown` rules in `frontend.css`. Do not force a plugin serif on published content. |

**Anti-goal:** Cloning FlexInvoice’s paper/ink/teal or Newsreader display into the WP block.

---

## 4. CSS / `editorStyle` approach (don’t break front-end)

### Current wiring (safe to keep)

- `block.json`: `"editorStyle": "file:../../assets/css/editor.css"` and `"style": "file:../../assets/css/frontend.css"`.
- `Assets.php` also enqueues `editor.css` on `enqueue_block_editor_assets` and `frontend.css` on the front when needed.

### Rules for the refresh

1. **All chrome / placeholder / PlainText / toolbar-adjacent polish** → `assets/css/editor.css` only (and selectors under `.bristlecone-markdown-editor` / block editor wrappers).
2. **Do not** move editor-only borders or min-heights onto bare `.bristlecone-markdown` in a way that changes published posts.
3. Preview in the editor currently uses **both** `.bristlecone-markdown-preview` and `.bristlecone-markdown`. Prefer:
   - Editor chrome (padding/border/min-height) on `.bristlecone-markdown-preview` / `.bristlecone-markdown-editor …`
   - Content typography/tables/TOC/etc. remain in `frontend.css` under `.bristlecone-markdown` (and legacy `.bits-markdown`).
4. Soften or remove the preview’s admin card border (`border: 1px solid #dcdcde`) so Preview feels like canvas content, not a boxed panel — without changing front-end TOC/table borders that live in `frontend.css`.
5. Kill textarea resize affordance via CSS (`resize: none`) if any fallback textarea remains.
6. Placeholder opacity ~0.62 (Jetpack / Gutenberg RichText convention).
7. No new SCSS pipeline; plain CSS only (matches “no build” constraint).

### Front-end regression checklist

- Published block still wraps with `.bristlecone-markdown`.
- Tables, mark, TOC, footnotes, page-break, bibliography, KaTeX/highlight enqueue behavior unchanged.
- Classic / document Markdown paths untouched by this ticket.

---

## 5. Impeccable — adopt process, not FlexInvoice tokens

### Finding (FlexInvoice)

Confirmed present in **`Zyniker13/FlexInvoice`** (reuse **process**, not palette):

| Path | Role |
| --- | --- |
| `PRODUCT.md` | Audience, jobs, constraints (impeccable product schema) |
| `DESIGN.md` | Normative visual system — **locks present** paper/ink/teal, Newsreader display |
| `docs/impeccable.md` | How agents use Impeccable beside `flexinvoice-ui` |
| `.cursor/skills/impeccable/` | Project-scoped skill (PR #46 era; installer: `npx impeccable install --providers=cursor --scope=project`) |
| `.cursor/skills/flexinvoice-ui/SKILL.md` | Product UI lane; points at DESIGN/PRODUCT |
| `.impeccable/design.json`, hooks | Detector / live helpers |

Account ops note: `Zyniker13/Documentation` → `cursor-and-bots/08-impeccable-design-skills.md` (pattern: complement product skills; trial via Cursor Manager; **not** a fit narrative for Docsify/ops — but a **scoped editor-surface** install in a WP plugin is reasonable if Mahler wants agent craft vocabulary).

**Caveat (steering):** FlexInvoice is Next/Tailwind SaaS. Bristlecone Markdown must get its **own** `DESIGN.md` from the block’s actual/target aesthetics (Jetpack-like typography-first, minimal chrome). **Do not** import paper/ink/teal or Newsreader rules.

### Recommendation for bits-markdown: **Adopt (lightweight)**

Worth doing so a later Cloud Agent does not invent “AI slop” chrome or trendy redesigns. Scope Impeccable to the **block editor writing surface**, not a full WP admin redesign.

#### Exact steps for Cursor Cloud Agent (after plan approval)

1. **Working copy:** clone/checkout `Zyniker13/bits-markdown` on a feature branch (e.g. `feat/block-aesthetics-1.2.0`).
2. **Install skill (project scope):**
   ```bash
   npx impeccable install --providers=cursor --scope=project
   ```
   Ensure `.cursor/skills/impeccable/` is committed (Cloud Agents see project skills the same way FlexInvoice does).
3. **Init product context (plugin-specific, not FlexInvoice copy):**
   - Run `/impeccable init` **or** hand-author `PRODUCT.md` at repo root capturing: WP plugin, Gutenberg Markdown block + Classic/document modes, iA Writer–aligned syntax, Jetpack replacement posture, plain `wp.*` JS / no build, audiences (writers, Jetpack migrants).
4. **Design system for *this* block (not FlexInvoice):**
   - Prefer **`/impeccable document` after** a first visual pass lands, **or** author a thin target `DESIGN.md` up front that states the north star (“typography-first Markdown canvas”), then refine with `document` once CSS/JS ship.
   - Location: **`DESIGN.md` + `PRODUCT.md` at bits-markdown repo root**; optional status note `docs/impeccable.md` mirroring FlexInvoice’s complement pattern.
   - Explicit anti-references in DESIGN.md: no Jetpack branding; no FlexInvoice paper/teal; no Inter-gradient SaaS; no heavy Component Library field chrome on the writing surface.
5. **Craft commands that fit this ticket** (in order):
   - `/impeccable shape` — confirm states in §2 before coding (if not already locked by this plan).
   - `/impeccable typeset` — placeholder vs source mono vs preview inheritance.
   - `/impeccable quieter` — strip box border, resize handle, excess min-height chrome.
   - `/impeccable distill` — fewer competing surfaces (toolbar + one writing surface).
   - `/impeccable polish` — shipping pass + alignment to DESIGN.md.
   - `/impeccable audit` / `critique` — a11y (aria-label on PlainText), contrast of placeholder opacity, keyboard.
   - **Avoid** unconstrained `/impeccable bolder` or trendy direction unless Mahler asks.
6. **`/impeccable live`:** skip for Cloud Agents (needs human browser picking); human smoke on `[test WordPress site]` or local WP instead.
7. **Do not** copy FlexInvoice `DESIGN.md` tokens or `flexinvoice-ui` patterns into this plugin.

If Mahler prefers **skip Impeccable** for now: still implement the PlainText/CSS refresh from this plan; add DESIGN/PRODUCT later. Process adoption is recommended but not a blocker for the visual fix.

---

## 6. Scope / non-goals

### In scope

- Editor writing-surface aesthetics for `bristlecone/markdown` (and shared `markdownEdit` used by compatibility aliases).
- `assets/js/block.js` + `assets/css/editor.css` (+ tiny i18n strings).
- Preserve Source / Preview / Insert image / REST preview / math render in preview.
- Version bump to **1.2.0**, changelog / `readme.txt` entry when implemented.

### Non-goals

- Cloning Jetpack branding, colors, icons, or block name.
- Requiring a JS build (webpack/@wordpress/scripts) unless PlainText/`wp.element` gaps appear — **not expected**; do not introduce a build “for aesthetics.”
- Changing league/commonmark parsing, storage, Classic Markdown, comments, Tools converter, or Jetpack adoption logic.
- Restyling front-end `.bristlecone-markdown` content (except accidental shared-selector fixes).
- Importing FlexInvoice visual tokens or Next/Tailwind stacks.
- Full Gutenberg RichText / block-transforms rewrite.
- Pixel-perfect Jetpack clone of preview typography scale (Jetpack’s `editor.scss` heading scale is optional polish, not required for 1.2.0).

---

## 7. Versioning note

Aesthetic feature → bump **b** (minor):

- **1.1.1 → 1.2.0**
- Update: plugin header `Version`, `BRISTLECONE_MARKDOWN_VERSION`, `readme.txt` Stable tag + changelog section.

Patch (1.1.2) would be wrong for a user-visible editor experience change.

---

## 8. Implementation phases + acceptance criteria + open questions

### Phase A — Design lock (no code / or DESIGN.md only)

- Confirm open questions below with Mahler.
- Optionally add root `PRODUCT.md` + draft target `DESIGN.md` + `docs/impeccable.md` (process only).

### Phase B — Editor surface (code)

1. Replace `TextareaControl` with `PlainText` in `markdownEdit`.
2. Add empty unselected placeholder branch (Jetpack-like).
3. Optionally: unselected-with-content → preview.
4. Rewrite `editor.css` for borderless source, placeholder, quieter preview chrome.
5. Keep toolbar + inspector behavior; polish pressed states if cheap.
6. Smoke: alias/compatibility path still edits.

### Phase C — Verify

- Block insert empty → placeholder matches approved copy/type.
- Select → borderless source, monospace, no resize grip.
- Preview tab → server HTML, math still renders, no heavy box.
- Save / reload / front-end unchanged for content CSS.
- No console errors; i18n strings wrapped in `__()`.
- Manual check on a theme with serif title (like Mahler’s screenshot) and a sans theme.

### Phase D — Ship hygiene

- Bump to **1.2.0**, changelog, PR via Cursor Manager after approval.
- If Impeccable installed: `/impeccable document` only if shipped look differs from draft DESIGN.md; commit `.cursor/skills/impeccable` if not already.

### Acceptance criteria

- [ ] Empty unselected block shows typography-first placeholder; **no** boxed textarea / resize handle.
- [ ] Selected Source uses borderless PlainText (or equivalent); monospace source.
- [ ] Source / Preview / Insert image still work; preview still server-driven.
- [ ] Front-end `.bristlecone-markdown` visual regressions absent.
- [ ] Still plain `wp.*` JS — no new mandatory build step.
- [ ] Version **1.2.0** with changelog note describing the editor aesthetic refresh.
- [ ] If Impeccable adopted: `PRODUCT.md` + Bristlecone-owned `DESIGN.md` present; FlexInvoice tokens absent.

### Open questions for Mahler

1. **Placeholder copy:** exact Jetpack string vs Bristlecone-specific wording?
2. **Unselected with content:** always show preview (Jetpack) or keep last tab (today)?
3. **Toolbar:** keep Dashicon Markdown/Preview or switch to labeled text tabs?
4. **Empty + Backspace deletes block?** (Jetpack yes)
5. **Impeccable install in bits-markdown now** (with own DESIGN.md) **or** aesthetics-only PR first?
6. **Serif on placeholder:** inherit theme only (recommended) or soft `ui-serif, Georgia, serif` fallback when theme is sans?
7. Any desire to quiet the Inspector PanelBody in the same pass?

---

## 9. Suggested Cursor Manager ticket text (AFTER plan approval only)

> **Title:** Bristlecone Markdown 1.2.0 — Gutenberg block aesthetic refresh (Jetpack-like writing surface)
>
> **Repo:** `Zyniker13/bits-markdown`  
> **Version:** bump **1.1.1 → 1.2.0** (aesthetic feature).  
> **Plan:** `[box scratch]/bristlecone-markdown-block-aesthetics-plan.md` (BITS WordPress Plugin Developer; Mahler-approved).
>
> **Goal:** Make the `bristlecone/markdown` editor feel typography-first and minimal-chrome (closer to Jetpack’s empty “Write your _Markdown_ **here**…” surface), without Jetpack branding and without a JS build.
>
> **Do:**
> 1. Replace `TextareaControl` with `wp.blockEditor.PlainText` in `assets/js/block.js`; add empty/unselected placeholder state; keep Source/Preview/Insert image + REST preview + math-in-preview.
> 2. Update `assets/css/editor.css` only for editor chrome (borderless source, placeholder opacity, quieter preview frame). Do not break front-end `.bristlecone-markdown` / `frontend.css`.
> 3. Optional (if Mahler approved Impeccable in open questions): `npx impeccable install --providers=cursor --scope=project`; add plugin-owned `PRODUCT.md` + `DESIGN.md` + short `docs/impeccable.md`. North star = Jetpack-like typography-first Markdown canvas. **Do not** copy FlexInvoice paper/ink/teal/Newsreader tokens. Prefer commands: `typeset`, `quieter`, `distill`, `polish` (± `audit`/`critique`). Skip unconstrained `bolder`.
> 4. Version bump to 1.2.0 + changelog / readme.txt.
>
> **Don’t:** Introduce @wordpress/scripts build unless blocked; change parsers/storage/Classic/comments/Jetpack adoption; redesign wp-admin; clone Jetpack brand.
>
> **Verify:** Empty placeholder; selected borderless mono source; Preview tab; front-end unchanged; theme inheritance smoke; PHPUnit still green.
>
> **PR:** Single focused PR to `main` after Cloud Agent run; BITS reviews against this plan.

---

## Appendix — code anchors (read-only)

**Bristlecone today (source control):**

```js
// assets/js/block.js — markdownEdit source branch
el( TextareaControl, {
  className: 'bristlecone-markdown-source',
  label: __( 'Markdown', 'bristlecone-markdown' ),
  hideLabelFromVision: true,
  value: markdown,
  onChange: setSource,
  rows: 16,
} )
```

```css
/* assets/css/editor.css */
.bristlecone-markdown-editor .bristlecone-markdown-source textarea { /* mono box */ }
.bristlecone-markdown-preview { border: 1px solid #dcdcde; /* … */ }
```

**Jetpack target pattern:**

```jsx
// edit.jsx (concept)
if ( ! isSelected && isEmpty ) {
  return <p className={`${className}__placeholder`}>
    { __( 'Write your _Markdown_ **here**…', 'jetpack' ) }
  </p>;
}
// else PlainText or MarkdownRenderer + Markdown/Preview toolbar tabs
```

```scss
// editor.scss (concept)
.wp-block-jetpack-markdown__placeholder { opacity: 0.62; pointer-events: none; }
.wp-block-jetpack-markdown__editor:focus { border-color: transparent; box-shadow: none; }
```

---

*End of plan. STOP — no plugin code changes in this phase.*
