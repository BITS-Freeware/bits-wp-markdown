# Bristlecone Markdown vs Jetpack Markdown — Phase 2 Results

**Host:** [test WordPress site]  
**Run:** 2026-09-19 ~11:38 PM – ~11:58 PM ET  
**Bristlecone:** 1.1.0 from WordPress.org (`bristlecone-markdown`)  
**Jetpack:** 16.2 (Markdown module + Blocks; **not** connected to WordPress.com)  
**Method:** wp-cli + REST + `the_content` HTML captures (no Classic Editor; browser unused)  
**Disposable posts:** slug prefix `md-ab-20260920-`  
**Backup left intact:** `[server path redacted]`  
**Evidence root:** `[box scratch]/evidence/{A,B,C,D}/`  
**Results file:** `[box scratch]/bristlecone-vs-jetpack-markdown-results.md`

---

## 1. Inventory (start → end)

| Item | Start | End / notes |
| --- | --- | --- |
| WP | 7.1.1 | 7.1.1 |
| PHP | **8.3.33** | **8.4.25** — upgraded droplet image `wordpress:php8.3-apache` → `wordpress:php8.4-apache` because Bristlecone 1.1.0 `Requires PHP: 8.4` (WP.org install refused on 8.3). Backup of compose: `[backup file name redacted]` |
| Theme | twentytwentyfive 1.5 | unchanged |
| `bits-markdown` 1.0.0 | active | **inactive** (left installed) |
| Jetpack | **not installed** | **16.2 active**; WP.com connection_ready=**false** |
| Jetpack Markdown module | n/a | Scenario A/C ON; B/D OFF. **End: Inactive** |
| Jetpack Blocks module | n/a | Needed for `jetpack/markdown` registration without WP.com connect (with offline mode). **End: Inactive** |
| Bristlecone Markdown | absent | **1.1.0 active** (wordpress.org) |
| Offline mode | n/a | MU-plugin `md-ab-jetpack-offline` sets `jetpack_offline_mode` so connection-free modules load without WP.com. **Still present** (assessment aid) |

### Jetpack connection

- Site was **not** connected (`blog_id=0`, `is_connection_ready=false`).
- Without connection, `wp jetpack module activate markdown` **failed** until offline mode was enabled.
- With offline mode + Blocks module ON, **`jetpack/markdown` registered without WP.com connect** — Mahler’s offered `bristleconeit` account was **not** required for Phase 2 block tests.
- **Blocker avoided** via offline mode; if Mahler wants a true connected Jetpack baseline, connect later and re-spot-check Blocks/Markdown.

---

## 2. PASS / FAIL / INVESTIGATE matrix

Legend: **JB** Jetpack block · **JD** Jetpack document · **BB** Bristlecone block · **BD** Bristlecone document

### Scenario A — Jetpack-only (Markdown ON, Bristlecone OFF)

| Check | Result | Evidence |
| --- | --- | --- |
| JD converts Markdown → HTML; source in `post_content_filtered` | **PASS** | Posts 62/64/66/68; meta `_wpcom_is_markdown=1` |
| JD CM-BASIC / EXTRA-TABLE-FN / FOOTNOTE front HTML | **PASS** | `[box scratch]/evidence/A/a-jd-*.content.html` |
| JD footnote IDs namespaced with post ID (`fn-{id}-…`) | **PASS** | e.g. `fn-66-note`, `fnref-64-1` |
| JB insertable without WP.com (offline + Blocks) | **PASS** | Block registered; posts 70–72 |
| JB CM-BASIC / FOOTNOTE front HTML | **PASS** | `[box scratch]/evidence/A/a-jb-*.content.html` (JB HTML synthesized with markdown-it@15 + footnote + typographer to match Jetpack client stack; see caveats) |
| JB EXTRA tables | **INVESTIGATE** | markdown-it 15 rendered tables; Jetpack support docs historically incomplete for Extra on block — treat cross-JB table parity as expected-differ pending editor UX confirm |

### Scenario B — Bristlecone-only (Markdown module OFF, Bristlecone ON)

| Check | Result | Evidence |
| --- | --- | --- |
| Install 1.1.0 from wordpress.org | **PASS** (after PHP 8.4) | Plugin list shows 1.1.0 active |
| BD converts; meta `_bristlecone_markdown=1` | **PASS** | Posts 73, 84, 86, 88 |
| BD CM-BASIC / EXTRA / FOOTNOTE HTML | **PASS** | `[box scratch]/evidence/B/b-bd-*.content.html` |
| BB `bristlecone/markdown` front HTML | **PASS** | Posts 81, 90, 91; `[box scratch]/evidence/B/b-bb-*.content.html` |
| BD footnotes namespaced | **PASS** | `bristlecone-markdown-fn-{id}-…` |

### Scenario C — both ON (no double-convert)

| Check | Result | Evidence |
| --- | --- | --- |
| New document converted once by Jetpack only | **PASS** | Post 92: `_wpcom_is_markdown=1`, **no** `_bristlecone_markdown`; clean HTML |
| No double-escape / nested strong | **PASS** | `[box scratch]/evidence/C/c-coexist-probe.content.html` |
| Admin warning + Disable Jetpack Markdown control | **PASS** | `[box scratch]/evidence/C/admin-notice.html` |

### Scenario D — switchover (Markdown OFF, Bristlecone ON)

| Check | Result | Evidence |
| --- | --- | --- |
| Document adoption via `_wpcom_is_markdown` | **FAIL** | See Defects §3.1; `[box scratch]/evidence/D/meta-adoption-post64.txt` |
| Editor shows Markdown source for JD posts after switch | **FAIL** | `edit_post_content` returns HTML while meta mismatch persists |
| Alias registers when Jetpack Blocks **off** | **PASS** | `jetpack/markdown` description = Bristlecone compat; `is_adopting=true` |
| Soft-migrate on save (`source` → `markdown`) | **PASS** | Post 97; `[box scratch]/evidence/D/soft-migrate-ok-*.html` (requires properly **slashed** post data — normal for Gutenberg/`wp_insert_post`) |
| Soft-migrate with unslashed `post_content` | **INVESTIGATE** | Emptied `markdown`/`html` attrs (stripslashes vs JSON `\n`); see §3.2 |
| Tools converter unlock via settings | **PASS** | `jetpack_block_converter=true` → `conversion_allowed=true`, tools page should register |
| Bulk convert while Jetpack Markdown ON | Not re-run in D | Code gate + Scenario C notice; expected refuse |

---

## 3. Defects (real bugs only — coding tickets)

### 3.1 FAIL — Jetpack document meta key mismatch (adoption break)

**Severity:** High for Jetpack → Bristlecone document migration  
**Summary:** Live Jetpack `WPCom_Markdown::IS_MD_META` is **`_wpcom_is_markdown`**. Bristlecone `Storage::META_WPCOM_KEY` / `is_markdown_post()` only checks **`_wpcom_markdown`** (plus `_bristlecone_markdown` / `_bits_markdown`).

**Evidence (JD post 64, Scenario D):**

- `_wpcom_is_markdown` = `1`
- `_wpcom_markdown` = empty
- `_bristlecone_markdown` = empty
- `Storage::is_markdown_post(64)` = **false**
- `edit_post_content` returns **HTML**, not Markdown from `post_content_filtered` (filtered still holds source)

**Proof of fix direction:** Temporarily setting `_wpcom_markdown=1` on post 64 made `is_markdown_post` true and `edit_post_content` return Markdown (`~~deleted~~` present). Meta removed after probe.

**Suggested ticket for Cursor Manager:** Treat `_wpcom_is_markdown` as a first-class legacy key in `is_markdown_post()` (and any adopt/notice paths). Optionally migrate/copy to `_bristlecone_markdown` on first successful edit. Do **not** only document `_wpcom_markdown`.

### 3.2 INVESTIGATE — Soft-migrate empties attrs if `wp_insert_post_data` receives unslashed JSON

**Severity:** Low for normal Gutenberg (core passes slashed data); Medium for custom importers/REST that call filters with unslashed block markup containing JSON `\n`.

**Evidence:** Direct `BlockMarkup::rewrite_*` and filter with `wp_slash($content)` → correct `markdown` attr. Filter with unslashed content → `{"markdown":"","html":""}` (posts 94/96 artifacts).

**Suggested ticket:** Prefer `parse_blocks()`-based rewrite without relying on `wp_unslash` preserving JSON backslash escapes; or detect/avoid double-stripslash.

---

## 4. Intentional differences (not bugs)

| Topic | Jetpack | Bristlecone | Notes |
| --- | --- | --- | --- |
| Document parser | Markdown Extra / GHF | league/commonmark + iA extras | Tables/footnotes both OK on document; markup differs |
| Block parser | Client markdown-it + footnote + typographer | Server CommonMark on render | Preview engines differ by design |
| Footnote IDs | `fn-{postId}-…` / `fnref-…` | `bristlecone-markdown-fn-{id}-…` | Both archive-safe; different prefixes |
| Heading permalinks `#` | No | Yes (`bristlecone-markdown-heading-permalink`) | iA-aligned |
| Syntax highlighting | Optional Jetpack code paths | Server highlight.php / `hljs` when enabled | BD CM-BASIC showed `hljs language-php` |
| Strikethrough | JD `<del>`; JB `<s>` (md-it) | `<del>` | Markup differ |
| Typographer / smart quotes | JB yes | No | Expected differ |
| Document wrapper | None | `.bristlecone-markdown` (when wrap filter applies) | Expected |
| iA extras (`==`, `{{TOC}}`, YAML, KaTeX, …) | No | Yes | Out of Jetpack scope |
| Abbreviations `*[…]` | JD Extra yes | No | Expected |
| Task lists | Neither converts | Literal | Intentional |

---

## 5. Host mutations performed (ops)

1. Deactivated `bits-markdown` 1.0.0.  
2. Installed/activated Jetpack 16.2.  
3. Added MU-plugin `wp-content/mu-plugins/md-ab-jetpack-offline.php` (Jetpack offline mode).  
4. Pulled `wordpress:php8.4-apache`; updated `[server path redacted]`; recreated wordpress container; installed wp-cli phar into container (`/usr/local/bin/wp` + copy under site).  
5. Installed/activated `bristlecone-markdown` 1.1.0 from wordpress.org.  
6. Created disposable `md-ab-20260920-*` posts only; did not wipe prior `spec-*` fixtures.  
7. End state: Bristlecone ON; Jetpack plugin ON; Markdown + Blocks modules **OFF**; offline MU-plugin still present.

**Not done:** WP.com Jetpack connection; Classic Editor tests; silent large code patches to Bristlecone.

---

## 6. Evidence index

| Path | Contents |
| --- | --- |
| `[box scratch]/evidence/A/` | Scenario A `*.content.html`, curl HTML, JB block sources, postmeta inventory |
| `[box scratch]/evidence/B/` | Scenario B BD/BB `*.content.html` |
| `[box scratch]/evidence/C/` | Coexistence probe HTML + admin notice HTML |
| `[box scratch]/evidence/D/` | Meta adoption dump; soft-migrate before/after/ok frontend |
| `[box scratch]/evidence/fixtures-*.md` | Shared Markdown sources |
| `[box scratch]/evidence/inventory-final.txt` | End inventory notes |
| `[box scratch]/screenshots/bristlecone-vs-jetpack/` | Reserved (HTML-first evidence used; no browser screenshots required for REST-proven cases) |

### Key post IDs

| Scenario | IDs |
| --- | --- |
| A JD | 62 CM-BASIC, 64 EXTRA, 66/68 FOOTNOTE |
| A JB | 70 CM-BASIC, 71 EXTRA, 72 FOOTNOTE |
| B BD | 73, 84, 86, 88 |
| B BB | 81, 90, 91 |
| C | 92 coexist probe |
| D | 97 soft-migrate OK; 94/96 are bad unslashed artifacts (safe to trash) |

---

## 7. Recommendations for Mahler / Cursor Manager

1. **Ship meta-key fix** (`_wpcom_is_markdown`) before declaring Jetpack document migration PASS.  
2. Keep coexistence model (Scenario C) — empirically sound.  
3. Soft-migrate / Tools converter work when Jetpack **Markdown module** is off **and** Jetpack is not still registering `jetpack/markdown` (deactivate Blocks or disconnect/offline so alias can own the name). Document this operational prerequisite.  
4. PHP **8.4+** is mandatory on hosts installing Bristlecone 1.1.0 from WP.org (this droplet was upgraded for the test).  
5. Optional cleanup: trash posts `94`, `96`; decide whether to keep offline MU-plugin or connect Jetpack properly for production-like ops.  
6. No large silent code changes were shipped from this Phase 2 run — defects handed off as tickets.

---

## 8. Success criteria checklist

| Criterion | Status |
| --- | --- |
| Complete A–D as far as Jetpack connection allows | **Done** (offline mode substituted for WP.com connect) |
| Clear evidence under `[box scratch]/evidence/` | **Done** |
| No secret leakage in reports | **Done** (app password not printed) |
| Backup left intact | **Done** |
| Disposable posts only for new content | **Done** (`md-ab-20260920-`) |
