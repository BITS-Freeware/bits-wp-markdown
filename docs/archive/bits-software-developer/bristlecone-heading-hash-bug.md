# Bristlecone Markdown — heading permalink `#` visible in heading text

**Status:** Confirmed bug (live + source)  
**Plugin:** Bristlecone Markdown (`Zyniker13/bits-markdown`), main @ 1.1.1  
**Live sample:** [live podcast post, URL redacted] (plugin CSS/assets reported as `ver=1.1.0` on that host)  
**Severity:** User-visible + SEO leak  
**Versioning:** Bug fix → bump **c** only (`1.1.1` → `1.1.2`). Not a feature/b release. May fold with other approved patch work later.

---

## 1. Symptom

Headings on published posts show an extraneous leading `#` glued to the title text:

- Visible / WebFetch: `#Information`, `#Show Notes`, `#Transcript`, `#Donate` (often rendered as `## # Information` by HTML→Markdown converters)
- Live DOM (2026-09-20 ET):

```html
<h2 id="information"><a href="#information" class="bristlecone-markdown-heading-permalink" aria-hidden="true" tabindex="-1" title="Permalink">#</a>Information</h2>
```

There is **no space** between the permalink `<a>` and the heading text, so the page reads as `#Information`.

**Collateral:** SEO meta on the same post includes the marker:

```text
<meta name="description" content="#Show Notes Forum Topic" />
```

`aria-hidden="true"` hides the `#` from some assistive tech, but scrapers, WebFetch, copy/paste, and meta generators still treat it as content. CSS only dims it (`opacity: 0.35`); it does **not** remove it from the text layer.

---

## 2. Root cause (one-liner)

`Parser.php` configures League CommonMark `HeadingPermalinkExtension` with `symbol => '#'` + `insert => INSERT_BEFORE`, so every heading gets a visible `#` character inside a permalink `<a>` that CSS only dims—not strips—leaving `#Heading` in HTML text and SEO.

### How heading permalinks are parsed / rendered

| Stage | File | Behavior |
| --- | --- | --- |
| Preprocess | `includes/Preprocessor.php` → `inject_heading_references()` | ATX headings get `{#slug}` / cross-ref labels; **does not** invent a leading `#` in heading text |
| Convert | `includes/Parser.php` (`heading_permalink` config + `HeadingPermalinkExtension`) | After parse, League inserts a `HeadingPermalink` node **before** heading children |
| Render | `vendor/league/.../HeadingPermalinkRenderer.php` | Emits `<a class="bristlecone-markdown-heading-permalink" …>#</a>` using config `symbol` |
| Explicit ids | `includes/Extension/Heading/HeadingIdProcessor.php` | Preserves author `{#id}`; rewrites permalink slug to match — unrelated to the visible `#` glyph |
| Style | `assets/css/frontend.css` | `.bristlecone-markdown-heading-permalink { opacity: 0.35; }` — hide-by-dim only |

Relevant config today (`includes/Parser.php` ~69–79):

```php
'heading_permalink' => array(
    'min_heading_level'   => 1,
    'max_heading_level'   => 6,
    'insert'              => HeadingPermalinkProcessor::INSERT_BEFORE,
    'id_prefix'           => '',
    'fragment_prefix'     => '',
    'apply_id_to_heading' => true,
    'html_class'          => 'bristlecone-markdown-heading-permalink',
    'symbol'              => '#',   // ← visible text source
    'title'               => 'Permalink',
    'aria_hidden'         => true,
),
```

### Why this is **not** an iA Writer `## #Heading` strip failure

- Live heading **text** is `Information` / `Show Notes` / … — there is no second literal `#` text node from Markdown source.
- Source of the glyph is exclusively the League permalink `symbol`.
- iA Writer docs (cross-refs / `[Label]` / `{{TOC}}`) do **not** require a visible `#` prefix on rendered headings. Plugin docs currently call the visible `#` “intentional” (`docs/plugin-home.md`); Mahler’s confirmed UX + SEO leak override that — treat as bug.

CSS-only hide is insufficient: meta description already contains `#Show Notes…`.

---

## 3. Minimal Markdown repro

Path: `[box scratch]/bristlecone-heading-hash-repro/heading-permalink-symbol.md`

```markdown
## Information

## Show Notes

### Donate
```

**Actual HTML shape (buggy, matches live):** see `expected-buggy.html` — `<a …>#</a>Information`.

**Expected HTML after fix:** see `expected-fixed.html` — empty symbol, ids + permalink `<a href="#…">` retained:

```html
<h2 id="information"><a href="#information" class="bristlecone-markdown-heading-permalink" aria-hidden="true" tabindex="-1" title="Permalink"></a>Information</h2>
```

Optional PHPUnit assertion sketch (add under `tests/ExtrasTest.php` or a new case):

```php
public function test_heading_permalink_symbol_not_visible_in_text(): void {
    $html = $this->html( "## Information\n" );
    $this->assertStringContainsString( 'id="information"', $html );
    $this->assertStringContainsString( 'bristlecone-markdown-heading-permalink', $html );
    $this->assertDoesNotMatchRegularExpression(
        '/class="bristlecone-markdown-heading-permalink"[^>]*>#/',
        $html
    );
    $this->assertStringContainsString( '>Information</h2>', $html );
}
```

(No local `php` on the investigation box; repro confirmed via live DOM + League renderer source.)

---

## 4. Suggested patch outline (do **not** open a PR from this investigation)

### Preferred fix (minimal, keeps id + clickable permalink)

1. **`includes/Parser.php`** — set `'symbol' => ''` in `heading_permalink` config.  
   League’s renderer already supports empty string (`Expect::string()`); it emits `<a …></a>` with no text content while keeping `href`, `id` on the heading (`apply_id_to_heading => true`), TOC, and cross-refs.

2. **`assets/css/frontend.css`** — optional cleanup:
   - Drop or rethink rules that assume a visible `#` glyph (margin/opacity for empty anchors).
   - If a hover affordance is desired later: `opacity: 0` by default + `h*:hover > .bristlecone-markdown-heading-permalink::before { content: "#"; }` (CSS-only glyph stays out of SEO text). Not required for the bugfix.

3. **Docs** — update `docs/plugin-home.md` (screenshot note that says the `#` before heading text is intentional) and any readme table rows that imply a visible `#` is iA-aligned.

4. **Tests** — add the assertion above; keep existing checks that the permalink **class** / `href="#slug"` still exist (`DocumentFixtureTest`, `ExtrasTest`).

5. **Version bump (c):** `1.1.1` → `1.1.2` in `bristlecone-markdown.php` (`Version` header + `BRISTLECONE_MARKDOWN_VERSION`), `readme.txt` stable tag / changelog.

### Alternatives (not preferred as sole fix)

| Approach | Pros | Cons |
| --- | --- | --- |
| `insert => INSERT_NONE` | No `<a>` at all; ids still applied | Loses clickable-in-heading permalink; weaker match to “keep permalink” |
| CSS `opacity: 0` / `font-size: 0` only | No PHP change | `#` remains in DOM → SEO/meta/WebFetch still broken |
| Custom `HeadingPermalinkRenderer` | Full control | Unnecessary if empty `symbol` works |

### Existing stored HTML (ops note)

Document-mode and block HTML already saved in `post_content` **will keep the `#` until reconverted**. Shipping 1.1.2 alone fixes **new saves / previews**. Plan either:

- Re-save affected posts (or a one-shot admin tool that re-runs Markdown → HTML from `post_content_filtered` / block source attributes), and/or  
- A narrow post-process strip: remove text node `#` inside `.bristlecone-markdown-heading-permalink` on already-rendered content (migration helper).

Podcasts sample headings live inside `wp-block-bristlecone-markdown` (some still aliased as `wp-block-jetpack-markdown`).

---

## 5. Cursor Manager ticket text (paste after approval)

**Title:** Fix visible heading permalink `#` (1.1.1 → 1.1.2)

**Body:**

> Confirmed on [live podcast post, URL redacted] headings render as `#Information`, `#Show Notes`, etc. Live HTML is `<h2 id="…"><a class="bristlecone-markdown-heading-permalink" …>#</a>Information</h2>`. Meta description also leaks `#Show Notes Forum Topic`.
>
> **Cause:** `includes/Parser.php` sets League `heading_permalink.symbol` to `'#'` with `INSERT_BEFORE`. `frontend.css` only dims the glyph (`opacity: 0.35`); it stays in the text/SEO layer. Not a Preprocessor strip bug; source heading text is clean.
>
> **Fix (bug → bump c to 1.1.2):**
> 1. Set `heading_permalink.symbol` to `''` in `Parser.php` (keep ids + permalink `<a href>`).
> 2. Adjust `frontend.css` if empty-anchor spacing needs it; optionally hover-only `::before` later.
> 3. Update `docs/plugin-home.md` (remove “visible `#` is intentional”).
> 4. Add PHPUnit: permalink class/href present, no `>#` inside the permalink anchor.
> 5. Note: existing stored HTML needs re-save/reconvert or a migration strip.
>
> Investigation notes: `[box scratch]/bristlecone-heading-hash-bug.md`  
> Minimal repro: `[box scratch]/bristlecone-heading-hash-repro/`
>
> Do not open a PR until Mahler approves. No feature/b bump.

---

## 6. Confidence

**High (0.92).** Live DOM matches League renderer + `Parser.php` config exactly; Preprocessor path ruled out; SEO leak confirms CSS-only approach is inadequate.
