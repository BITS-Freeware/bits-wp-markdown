# Bristlecone Markdown vs Jetpack Markdown — Assessment & Live Test Plan

**Phase:** 1 (research + plan only)  
**Status:** Awaiting Mahler approval before any install, mutation, or live A/B runs  
**Prepared:** 2026-09-19 11:27 PM ET  
**Bristlecone under assessment:** `Zyniker13/bits-markdown` @ `9875fd0` (main / **1.1.0**) — WP.org slug `bristlecone-markdown`  
**Jetpack under assessment:** Automattic/jetpack trunk (Markdown module + `jetpack/markdown` block as of research date)

---

## Confirmed live A/B host

| Item | Value |
| --- | --- |
| **Test install URL** | **[test WordPress site]** |
| Host / stack | Docker WordPress on droplet `[redacted IP]`, stack path `[server path redacted]` |
| Confirmed by | WordPress Server Manager |
| Phase 1 constraint | **Do not log in, install plugins, create posts, or mutate this site until Mahler approves Phase 2** |

### Access the live phase will need (credentials not collected in Phase 1)

- WP Admin URL: `[test WordPress site]/wp-admin/`
- Administrator account (or Application Password) able to: install/activate plugins, toggle Jetpack modules, edit Settings → Writing / Discussion, create posts/pages/comments, use Tools pages
- Optional SSH to droplet `[redacted IP]` (stack `[server path redacted]`) for DB spot-checks of `post_content` / `post_content_filtered` / postmeta — only if Mahler authorizes
- Jetpack: site connection status known (block may require Jetpack connection per Jetpack docs; Classic Markdown module does **not** require connection)

---

## 1. Scope — what “full assessment” covers

A full assessment means code-backed expectations plus (after approval) controlled live A/B on `[test WordPress site]`. It does **not** mean claiming pixel-identical HTML across parsers.

### In scope

| Area | What we assess |
| --- | --- |
| **Syntax parity** | CommonMark baseline; Jetpack Markdown Extra (document) / markdown-it CommonMark+footnotes (block); Bristlecone CommonMark + iA Writer extras |
| **Storage model** | `post_content` HTML vs `post_content_filtered` Markdown; meta flags; block attributes; deactivate-safe HTML |
| **Block editor UX** | Insert / edit / preview; missing-block behavior; soft-migrate on save; default-to-Markdown setting |
| **Classic / document mode** | Classic Editor Text tab, REST body, iA Writer Publish path (Bristlecone); Jetpack Classic Markdown module |
| **Comments** | Optional Markdown comments (both); sanitization strictness |
| **Front-end output** | Rendered HTML structure, CSS wrappers, footnotes on single + archive, math/highlight assets |
| **Coexistence / migration** | Jetpack module on vs off with Bristlecone active; adopt document posts; alias `jetpack/markdown` → `bristlecone/markdown`; Tools converter |
| **Settings** | Post types, comments, highlighting, math, default-to-Markdown, Tools converter unlock |
| **Known intentional differences** | Documented as Expected differ (not bugs) |

### Out of scope (unless Mahler expands)

- Replacing Jetpack as a whole (only Markdown module + Markdown block)
- Full iA Writer app parity (library, Content Blocks / transclusion, typography, export)
- Performance / load testing
- WP.com hosted Markdown vs self-hosted Jetpack (we test self-hosted Jetpack on the droplet)
- Security penetration testing beyond XSS fixtures already in Bristlecone’s suite

### Success definition (after Phase 2)

Mahler can decide with evidence whether Bristlecone is a safe Jetpack Markdown replacement on this host: syntax gaps are catalogued, storage/migration paths work, and coexistence does not double-process.

---

## 2. Architecture differences (from code)

### 2.1 High-level

| Concern | Jetpack document module | Jetpack Markdown block | Bristlecone (block + document) |
| --- | --- | --- | --- |
| **Entry** | Module `markdown` → `modules/markdown/easy-markdown.php` → `WPCom_Markdown` | Extension block `projects/plugins/jetpack/extensions/blocks/markdown/` | Plugin bootstrap `bristlecone-markdown.php` → `Plugin::init()` |
| **Parser** | PHP **Markdown Extra** (`MarkdownExtra_Parser` + `WPCom_GHF_Markdown_Parser` in `_inc/lib/markdown/`) | Client **markdown-it** + `markdown-it-footnote`; `typographer: true` | Server **league/commonmark** ^2.7 (2.10) + extras + custom `MarkdownExtension` |
| **When it runs** | On `wp_insert_post_data` if posting option on + `post_type_supports( …, 'wpcom-markdown' )`; **skips if `has_blocks()`** | At **save** via `save.jsx` → client HTML into post content; editor preview also client-side | Document: `Storage` on `wp_insert_post_data` when Jetpack Markdown **inactive**. Block: PHP `render_callback` + REST `bristlecone-markdown/v1/preview` |
| **Preview** | Classic: no shared live preview (Visual can mangle); o2 helpers exist | Client markdown-it (same as save path) | Server PHP via REST — **same engine as publish** |

### 2.2 Storage keys & block attrs

| Item | Jetpack document | Jetpack block | Bristlecone document | Bristlecone block |
| --- | --- | --- | --- | --- |
| HTML | `post_content` | Inner HTML from client render (saved markup) | `post_content` | Attr `html` + innerHTML fallback; `render_callback` re-parses `markdown` when present |
| Markdown source | `post_content_filtered` | Attr **`source`** | `post_content_filtered` | Attr **`markdown`** (+ `html`) |
| Flag meta | **`_wpcom_is_markdown`** (`WPCom_Markdown::IS_MD_META`) | (none beyond block presence) | **`_bristlecone_markdown`** (also reads legacy `_bits_markdown`) | N/A (block attrs) |
| Front matter | N/A | N/A | `_bristlecone_markdown_front_matter` | Via parser when converting |
| Options | `wpcom_publish_posts_with_markdown`, `wpcom_publish_comments_with_markdown` | Jetpack Blocks suite (independent of Classic toggle) | `bristlecone_markdown_settings` | Same option bag |

**Critical code discrepancy (must live-verify):** Bristlecone docs/code constant `META_WPCOM_KEY = '_wpcom_markdown'`, but current Jetpack `WPCom_Markdown` sets **`_wpcom_is_markdown`**. Adoption / “is Markdown post?” detection may fail unless something else bridges this. Phase 2 must inspect postmeta on a Jetpack-created document post on `[test WordPress site]`.

### 2.3 Block names & render paths

| | Jetpack | Bristlecone |
| --- | --- | --- |
| Block name | `jetpack/markdown` | `bristlecone/markdown` |
| Inserter | Always available as Jetpack block (docs: remains available even if Classic Markdown module is off) | Own block; aliases register with `inserter: false` |
| Attr | `source` | `markdown`, `html` |
| Save | Client markdown-it HTML into block | Saves `markdown` + optional `html`; front-end prefers re-convert from `markdown` |
| Soft-migrate | — | When Jetpack Markdown module inactive and block not registered by Jetpack: register alias, soft-rewrite to `bristlecone/markdown` on save (`source` → `markdown`) |
| Bulk convert | — | Tools → Bristlecone Markdown Converter (settings unlock, default off) |

### 2.4 Coexistence gate (Bristlecone)

```text
Plugin::init():
  if Jetpack Markdown module active:
    do NOT register Storage / Rest / Comments
  else:
    register Storage, Rest, Comments (if enabled)
  always: Block, JetpackMarkdownBlock (aliases), Settings, notices
```

Bulk Jetpack→Bristlecone block conversion **no-ops** while Jetpack Markdown module is still active.

### 2.5 Document-mode wrappers / wpautop

- **Bristlecone:** wraps document HTML in `<div class="bristlecone-markdown">`; disables `wpautop` for flagged Markdown posts (footnotes).
- **Jetpack document:** historically relies on Extra + `strip_paras` / WordPress paragraph handling; footnote IDs rewritten (`fn:` → `fn-`). Does not use Bristlecone’s wrapper class.

### 2.6 Intentional Bristlecone extras (not Jetpack)

From `Parser.php` + `MarkdownExtension` + `Preprocessor`: highlight `== ==`, GFM tables, autolink, description lists, attributes, footnotes (named + inline with spaces), heading permalinks `#`, `{{TOC}}`, YAML + `[%key]`, KaTeX `$`/`$$`, sup/sub, `+++` page break, `//` writer comments, MultiMarkdown citations, server `highlight.php`, DisallowedRawHtml + KSES.

**Intentionally not converted:** task lists `- [ ]` / `- [x]`; iA Writer Content Blocks / file transclusion.

---

## 3. Feature / syntax matrix

Legend for columns **JB** = Jetpack block · **JD** = Jetpack document module · **BB** = Bristlecone block · **BD** = Bristlecone document  

**Expect:** Same · Differ · Need live verify  

| Feature / syntax | JB | JD | BB | BD | Expect | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| ATX headings `#` … `######` | Y | Y | Y | Y | Same (core) | Jetpack GFM: `#` without space after hash often **not** a heading |
| Setext headings | Y* | Y | Y | Y | Need live verify | Bristlecone: weaker auto cross-ref for setext |
| Emphasis / strong | Y | Y | Y | Y | Same | |
| Links / images | Y | Y | Y | Y | Same | |
| Lists (ul/ol) | Y | Y | Y | Y | Same | |
| Blockquotes | Y | Y | Y | Y | Same | |
| Fenced code \`\`\` / ~~~ | Y | Y | Y | Y | Differ (highlight) | Bristlecone optional server hljs classes; Jetpack may use `[code]` shortcode if SyntaxHighlighter present |
| Indented code | Y | Y | Y | Y | Need live verify | |
| Thematic break `---` / `***` | Y | Y | Y | Y | Differ edge | Bristlecone: leading YAML `---` and page-break `+++` can steal CommonMark thematic-break cases |
| Strikethrough `~~` | N† | Y | Y | Y | Differ (JB) | †markdown-it core off unless plugin; Extra/GHF has strikethrough; Bristlecone extension on |
| Tables (pipe) | N† | Y (Extra) | Y | Y | Differ (JB) | Jetpack issue #18035 lists tables-like extras incomplete on block |
| Autolink bare URLs | N† | Partial | Y | Y | Differ | |
| Footnotes `[^1]` | Y (plugin) | Y (Extra) | Y | Y | Differ markup | ID prefixes: Jetpack `{postId}-` / `fn-`; Bristlecone `bristlecone-markdown-fn-{id}-` |
| Inline footnotes `[^text with spaces.]` | Need verify | Need verify | Y | Y | Differ | Bristlecone iA Writer–aligned |
| Definition / description lists | N† | Y (Extra) | Y | Y | Differ (JB) | |
| Abbreviations `*[HTML]:` | N | Y (Extra) | N | N | Differ | Jetpack Extra only |
| Highlight `==text==` | N | N | Y | Y | Differ | Bristlecone / iA Writer |
| Math `$` / `$$` (KaTeX) | N | Legacy `$latex …$` preserve only | Y | Y | Differ | Bristlecone setting; Jetpack preserves old latex shortcode paths, does not KaTeX `$`/`$$` |
| Superscript / subscript | N† | N | Y | Y | Differ | |
| YAML front matter | N | N | Y | Y | Differ | Maps title/excerpt/tags/categories/slug |
| `[%key]` interpolation | N | N | Y | Y | Differ | |
| `{{TOC}}` | N | N | Y | Y | Differ | |
| Heading permalink `#` before heading | N | N | Y | Y | Differ | Class `bristlecone-markdown-heading-permalink` |
| Cross-ref `[Heading][]` / `{#id}` | N | Attrs Extra partial | Y | Y | Differ | |
| Page break `+++` | N | N | Y | Y | Differ | Visual HR-like; **not** `<!--more-->` |
| Writer comments `//` | N | N | Y | Y | Differ | Stripped from HTML |
| Citations `[p. 23][#Key]` | N | N | Y | Y | Differ | |
| Task lists `- [ ]` | N† | N | Literal | Literal | Same (neither converts) | Bristlecone intentional |
| Typographer (smart quotes) | Y (`typographer: true`) | N | N | N | Differ | Jetpack **block** only |
| Raw HTML (restricted) | Limited | Extra-friendly | DisallowedRawHtml + KSES | Same | Differ | Compare XSS fixtures |
| Comments Markdown | N (block N/A) | Optional Discussion setting | Optional settings | Same parser | Need live verify | Both use `pre_comment_content` priority 9 pattern |
| Soft line breaks | Need verify | WP.com notes differ from CommonMark | CommonMark | CommonMark | Need live verify | |

\*† = Jetpack block is CommonMark + footnotes plugin only; many GFM/Extra features are **Expected differ** vs document module and vs Bristlecone.

### Reusable Bristlecone fixtures (for A/B samples)

Under `[box scratch]/bits-markdown/tests/fixtures/`:

- **documents:** `kitchen-sink.md`, `kitchen-sink-page.md`, `footnotes.md`, `ia-writer-publish.md`, `commonmark-torture.md`, `unsafe-html.md`, `negative-ia-writer.md`, YAML samples, etc. (18 files)
- **blocks:** `jetpack-markdown.html`, `simple-markdown.html`
- **comments:** `basic.md`, `table-footnote.md`, `math.md`, `xss.md`

---

## 4. Live test-install protocol (AFTER Mahler approves)

**Host:** [test WordPress site]  
**Do not execute this section in Phase 1.**

### 4.1 Prerequisites

1. Confirm WP version ≥ 6.4 and PHP ≥ 8.4 (Bristlecone requirement).
2. Note active theme (CSS diffs are theme-sensitive).
3. Inventory: Jetpack installed? Connected? Markdown **module** on/off? Classic Editor plugin?
4. Create a dedicated admin user or use provided Application Password; record identity in evidence log (not passwords in the plan file).
5. Prefer disposable posts with slug prefix `md-ab-YYYYMMDD-` so cleanup is easy.
6. Snapshot or export before bulk Tools converter runs (if any real content exists).

### 4.2 Scenario matrix (plugin states)

Run fixtures under these **named** configurations (toggle one variable at a time):

| ID | Jetpack Markdown module | Bristlecone active | Intent |
| --- | --- | --- | --- |
| **A** | ON | OFF | Jetpack-only baseline (document + block) |
| **B** | OFF | ON | Bristlecone-only baseline |
| **C** | ON | ON | Coexistence: Bristlecone must **not** convert documents/comments; notice + disable control |
| **D** | OFF → then Bristlecone | ON | Switchover: prior Jetpack documents + `jetpack/markdown` blocks |

Optional **E:** Jetpack fully deactivated (not just Markdown module) — only if Mahler wants “no Jetpack” end state.

### 4.3 A/B fixtures (same Markdown everywhere)

Use identical source strings for block and document modes.

**Fixture pack (minimum):**

1. **CM-BASIC** — headings, emphasis, lists, links, image, blockquote, HR, fenced code  
2. **CM-EDGE** — nested lists, reference links, indented code (from CommonMark torture / `commonmark-torture.md`)  
3. **EXTRA-TABLE-FN** — pipe table + `[^1]` footnotes + strikethrough  
4. **IA-EXTRAS** — `==highlight==`, `{{TOC}}`, YAML front matter, `$E=mc^2$`, `$$…$$`, `^2`, `x~i~`, `+++`, `// comment`, citation  
5. **NEGATIVE** — task lists; Content Block–like paths; unsafe HTML (`xss.md` / `unsafe-html.md`)  
6. **FOOTNOTE-ARCHIVE** — two short posts with footnotes for archive collision check  

**Modes per fixture:**

- Block editor: one post, single Markdown block (Jetpack or Bristlecone per scenario)  
- Document: Classic Editor **Text** tab (or REST POST with raw Markdown body) for enabled post types  

### 4.4 Editor checks

| Check | How | Pass criteria |
| --- | --- | --- |
| Missing-block | Scenario D: open old `jetpack/markdown` post with Jetpack Markdown off + Bristlecone on | No “block is not available”; source editable |
| Soft-migrate | Edit + Update | Block name becomes `bristlecone/markdown`; attr `markdown` populated from `source`; front-end OK |
| Default-to-Markdown | Settings → enable; New Post | Starts with empty Bristlecone Markdown block; new block preference Markdown (not paragraph) |
| Preview parity (Bristlecone) | Toggle Preview tab | Preview HTML matches front-end (same PHP parser) |
| Preview vs front (Jetpack block) | Preview then publish | Expect client markdown-it; note any typographer/footnote differences vs Bristlecone |
| Coexistence notice | Scenario C | Warning + “Disable Jetpack Markdown” control; no double HTML |
| Comments | Enable both sides independently | Markdown converts once; XSS stripped |

### 4.5 Front-end HTML/CSS diff approach

1. Publish fixture posts; open singular URL and (for footnotes) an archive/home that lists both.  
2. Capture **View Source** or `curl -sL` of the content area into `[box scratch]/screenshots/` or an evidence folder:  
   `evidence/A/CM-BASIC.html`, `evidence/B/CM-BASIC.html`, …  
3. Diff semantically (not byte-identical): headings, links, footnote `href`/`id`, table structure, presence of wrappers (`.bristlecone-markdown` vs Jetpack block classes).  
4. Screenshots: editor source, editor preview, front-end (desktop). Store under `[box scratch]/screenshots/bristlecone-vs-jetpack/`.  
5. Note CSS: theme `sup` rules, `.hljs`, KaTeX enqueue only when math present.

### 4.6 Migration / Tools converter checks

1. Seed several posts containing `<!-- wp:jetpack/markdown -->` (use `tests/fixtures/blocks/jetpack-markdown.html` as template).  
2. With converter **off**: soft-migrate one post via Save only.  
3. Enable Settings → “Enable Jetpack Markdown block converter under Tools”.  
4. Tools → Bristlecone Markdown Converter: dry-read UI, confirm checkbox, run batch; verify counts and that Jetpack-active state blocks Jetpack conversion.  
5. Optional scan for other `*markdown*` unregistered blocks — confirm no auto-register.  
6. Document posts: after Scenario D, edit a Jetpack Classic Markdown post — **must show Markdown source**, not only HTML. Record actual postmeta keys (`_wpcom_is_markdown` vs `_wpcom_markdown` vs `_bristlecone_markdown`).

### 4.7 Pass / fail criteria & evidence

| Result | Meaning |
| --- | --- |
| **PASS** | Behavior matches Expected same / Expected differ; migration works; no double-processing in C |
| **FAIL** | Unexpected broken front-end, missing block, lost Markdown source, double-processed HTML, XSS executed |
| **INVESTIGATE** | Meta-key mismatch, theme CSS, or Jetpack connection gating the block |

**Evidence per fixture × scenario:** screenshot(s), HTML snippet (content div), post ID, plugin state checklist, short notes.  
**Do not** paste credentials into evidence files.

### 4.8 Suggested Phase 2 order (minimize churn)

1. Scenario **A** Jetpack-only baselines  
2. Scenario **B** Bristlecone-only baselines (install 1.1.0 from WP.org or release zip)  
3. Scenario **C** coexistence  
4. Scenario **D** switchover + Tools converter  
5. Write comparison results doc (Phase 3 deliverable — separate from this plan)

---

## 5. Risks / false comparisons

| Risk | Why it misleads | Mitigation |
| --- | --- | --- |
| **Client Jetpack preview vs server Bristlecone** | Jetpack block = markdown-it in browser; Bristlecone preview = PHP REST | Compare each product to its own front-end first; cross-compare published HTML only |
| **Theme CSS** | Footnotes/`sup`/tables look “wrong” for visual reasons | Diff DOM structure; note theme; optional minimal theme check |
| **Jetpack still active (double-processing)** | If both converted documents, mangled HTML | Scenario C must prove Bristlecone Storage is off while module on |
| **Jetpack block ≠ Jetpack document** | Extra vs CommonMark; tables/footnotes differ inside Jetpack itself | Never treat JB and JD as one “Jetpack Markdown” |
| **Typographer / smart punctuation** | Jetpack block softens quotes; Bristlecone does not | Mark Expected differ |
| **Meta key name mismatch** | Docs say `_wpcom_markdown`; Jetpack code uses `_wpcom_is_markdown` | Live postmeta inspect before declaring adoption PASS |
| **Jetpack clears `post_content_filtered` when module off and post re-saved without Bristlecone** | Easy to destroy Markdown source during switchover | Keep Bristlecone active before re-saving Jetpack documents; backup first |
| **has_blocks short-circuit** | Both skip whole-document conversion on block posts | Test Classic/REST separately from Gutenberg |
| **SyntaxHighlighter / latex modules** | Jetpack GFM may emit `[code]` or preserve `$latex` | Record active Jetpack modules on the test host |
| **Archive footnotes** | Multiple posts’ footnote IDs | Use Bristlecone namespacing check; compare Jetpack prefixes |
| **Comparing to WP.com support docs** | Self-hosted Jetpack may differ slightly | Prefer code + live host over marketing copy |

---

## 6. Open questions for Mahler

1. **Test install URL** — **RESOLVED:** `[test WordPress site]` (Docker WP, droplet `[redacted IP]`, `[server path redacted]`).  
2. **Credentials / access for Phase 2** — WP admin (and optional SSH) to be provided when approving Phase 2; not stored in this plan.  
3. **Scenario set** — Approve A/B/C/D as above? Add **E** (Jetpack fully removed)?  
4. **Jetpack connection** — Is Jetpack already connected on this host so `jetpack/markdown` is insertable?  
5. **Classic Editor** — Install Classic Editor plugin for JD tests, or use Classic block / REST only?  
6. **Content policy** — Is the site empty/disposable, or must we avoid touching existing posts?  
7. **Meta-key bug** — If live confirms `_wpcom_is_markdown` and Bristlecone only reads `_wpcom_markdown`, prioritize a code fix before declaring migration PASS?  
8. **WP.org vs GitHub zip** — Install Bristlecone **1.1.0** from wordpress.org or from `composer release` zip built from `9875fd0`?  
9. **iA Writer** — Include live Publish via Application Password, or defer REST simulation with `curl`?  
10. **Evidence location** — Prefer `[box scratch]/screenshots/…` only, or also commit/attach to a BITS ticket / Omnifora topic?

---

## Appendix A — Source map (research)

### Bristlecone (`Zyniker13/bits-markdown` @ 9875fd0 / 1.1.0)

- `bristlecone-markdown.php`, `README.md`, `readme.txt`, `docs/plugin-home.md`
- `includes/Parser.php`, `Storage.php`, `Block.php`, `JetpackCompat.php`, `JetpackMarkdownBlock.php`, `Settings.php`, `Comments.php`, `Plugin.php`, `Extension/MarkdownExtension.php`, `Preprocessor.php`
- `blocks/markdown/block.json` → `bristlecone/markdown` attrs `markdown`, `html`
- WP.org: https://wordpress.org/plugins/bristlecone-markdown/ (listed; Stable 1.1.0)

### Jetpack (Automattic/jetpack)

- Module: `projects/plugins/jetpack/modules/markdown.php` + `markdown/easy-markdown.php` (`WPCom_Markdown`)
- Parser: `_inc/lib/markdown.php` → Extra + `gfm.php` (`WPCom_GHF_Markdown_Parser`)
- Block: `extensions/blocks/markdown/` — `block.json` (`source`), `edit.jsx`, `save.jsx`, `renderer.jsx` (markdown-it + footnote), `markdown.php`
- Docs: https://jetpack.com/support/markdown/ (block); https://jetpack.com/support/markdown-classic-editor/ (document / Extra)

### Storage compatibility summary

Both document modes: HTML in `post_content`, Markdown in `post_content_filtered`, skip conversion when `has_blocks()`. Bristlecone additionally namespaces footnotes, wraps output, disables `wpautop`, and refuses document conversion while Jetpack’s Markdown **module** is active.

---

## Appendix B — Phase gate

| Phase | Action | Gate |
| --- | --- | --- |
| **1 (this doc)** | Research + plan | Done when Mahler reviews |
| **2** | Live A/B on [test WordPress site] | Explicit Mahler approval + credentials |
| **3** | Results report (pass/fail matrix, HTML diffs, recommendations) | After Phase 2 |

**Stop here until approved. No live login, plugin install, or content mutation in Phase 1.**
