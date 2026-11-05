# Bristlecone menu coexistence smoke — 2026-09-27

**Site:** [test WordPress site] (Docker WP test)  
**When:** 2026-09-27 ~21:18–21:24 EDT  
**Tester:** Grok Bot (BITS Plugin Developer smoke)  
**Mahler:** not messaged (per brief)

## Installs

| Plugin | Source | Version on disk | Method |
|--------|--------|-----------------|--------|
| Bristlecone Markdown | `https://github.com/Zyniker13/bits-markdown` main zipball `a00dc1d` | **1.3.0** | SSH → `[server path redacted]` (replaced 1.2.0). GitHub zip lacks Composer `vendor/`; ran `composer install --no-dev` on box before deploy. |
| Bristlecone Admin Styles | `https://github.com/Zyniker13/bits-wp-admin-styles` main zipball `d0a9dca` | **1.0.0** | SSH extract to `bristlecone-admin-styles/` (new). |

**Runtime:** WordPress 7.1.2, PHP 8.4.25 (`wordpress:php8.4-apache`).  
**Final plugin state (left after smoke):** both Bristlecone plugins **active**. Old inactive `bits-markdown` 1.0.0 left untouched. Jetpack remains active (unrelated).

---

## Check A — Markdown alone (Admin Styles deactivated)

| Criterion | Result | Evidence |
|-----------|--------|----------|
| Top-level **Bristlecone** menu | **PASS** | `#toplevel_page_bristlecone` present once |
| Bold **B** icon | **PASS** | Shared SVG data-URI menu icon (path forms capital B); visible white B in screenshots |
| Submenu **Markdown** opens settings | **PASS** | `admin.php?page=bristlecone-markdown` → HTTP 200, `<h1>Bristlecone Markdown</h1>`, no fatals |
| Old Settings → Bristlecone Markdown gone / redirect | **PASS** | Settings submenu has no Bristlecone/Markdown entry. `options-general.php?page=bristlecone-markdown` → **302** → `admin.php?page=bristlecone-markdown` |
| No fatals | **PASS** | No critical/parse/fatal markers on dashboard or settings |

**Screenshots**
- `[box scratch]/bristlecone-smoke/screenshots/A-dashboard-menu.png` — Dashboard; Bristlecone + Markdown flyout + bold B
- `[box scratch]/bristlecone-smoke/screenshots/A-markdown-alone.png` — Markdown settings with Admin Styles off

**HTML captures:** `A-dashboard.html`, `A-markdown.html`, `A-settings-menu.html`

---

## Check B — Both active

| Criterion | Result | Evidence |
|-----------|--------|----------|
| Single Bristlecone parent (no duplicate) | **PASS** | `bristlecone_parent_count=1` (WP-CLI menu dump + HTML) |
| Submenus **Admin Styles** and **Markdown** | **PASS** | Order: Admin Styles, then Markdown |
| Admin Styles settings loads | **PASS** | `admin.php?page=bristlecone-admin-styles` → 200, `<h1>Bristlecone Admin Styles</h1>` |
| Markdown settings loads | **PASS** | same URL as A → 200, h1 present |
| No fatals | **PASS** | No critical/parse/fatal markers |

**Screenshots**
- `[box scratch]/bristlecone-smoke/screenshots/B-both-admin-styles.png` — Admin Styles page; both submenus under one parent
- `[box scratch]/bristlecone-smoke/screenshots/B-both-markdown.png` — Markdown page with both active
- Note: `A-markdown-menu.png` was captured while both were still active (superseded by `A-markdown-alone.png` for Check A)

**HTML captures:** `B-dashboard.html`, `B-admin-styles.html`, `B-markdown.html`

---

## Check C — Optional

| Criterion | Result | Evidence |
|-----------|--------|----------|
| Admin Styles **enable** off by default | **PASS** | Option `bristlecone_admin_styles_settings` = `{"enabled":false,"font_family":"default","scopes":["edit-post"]}`; settings checkbox unchecked in UI |
| Posts list still loads | **PASS** | `edit.php` → 200, `<h1 class="wp-heading-inline">Posts</h1>`, list table renders, no fatals; Bristlecone menu still single parent with both submenus |

**Screenshot:** `[box scratch]/bristlecone-smoke/screenshots/C-posts.png`  
**HTML:** `C-posts.html`

---

## Summary

| Check | Outcome |
|-------|---------|
| A Markdown alone | **PASS** |
| B Both active coexistence | **PASS** |
| C Enable default + Posts list | **PASS** |

**Overall: PASS** — shared Bristlecone parent menu coexistence works for Markdown 1.3.0 + Admin Styles 1.0.0.

## Blockers

None for the smoke criteria.

**Notes (non-blocking)**
1. GitHub main zipball for Markdown does **not** ship `vendor/`; release/smoke deploys need `composer install --no-dev` (or a built release artifact). No GitHub Releases existed at test time.
2. WP REST `PUT /wp/v2/plugins/{plugin}` returned Apache 404 behind this stack; activation used WP-CLI in the WordPress container instead. REST `GET /wp/v2/plugins` worked.
3. Temporary WP auth cookies used for admin HTML/screenshots were deleted after the run; Application Password was not printed in this report.

## Site delta (smoke only)

- Upgraded `bristlecone-markdown` 1.2.0 → 1.3.0 (files on disk).
- Installed + activated `bristlecone-admin-styles` 1.0.0.
- No production content/posts edited; settings options only auto-created by plugin activation (Admin Styles enable left **false**).
