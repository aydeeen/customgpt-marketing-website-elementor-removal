# Elementor Removal — Project Spec

**Branch:** `dev`
**Goal:** Replace every Elementor-rendered surface on customgpt.ai with custom PHP templates, scoped CSS, and vanilla JS — eliminating the Elementor dependency from the frontend entirely.
**Approach:** One surface at a time, header first. Each surface ships independently so the site is never partially broken.

---

## Where the code lives

**All implementation files are in the main theme repo:**
[github.com/aydeeen/customgpt](https://github.com/aydeeen/customgpt) — branch `dev`

This repo is documentation only: project spec, architecture decisions, test log, and phase tracking. No code lives here.

---

## Why we're doing this

- Elementor adds ~500 KB of CSS/JS to every page load even when a page uses only a fraction of its features.
- The Elementor header and page templates cannot be version-controlled with the same precision as custom PHP.
- Custom templates are faster, easier to audit, and owned entirely by us.

---

## Current status

### ✅ Phase 1 — Custom header (complete, on `dev`)

Elementor header template (post ID 59911) replaced with a fully custom PHP/CSS/JS header.

**Files:**
| File | Purpose |
|---|---|
| `templates/template-elementor-header.php` | PHP markup — injected via `wp_body_open` at priority 1 |
| `assets/css/header.css` | All header styles — scoped to `#cgpt-header` |
| `assets/js/header.js` | All header interactions — vanilla JS, no jQuery |
| `functions.php` | Hook registration, menu registration, menu data helpers |

**Architecture decisions:**

- **Injection point:** `wp_body_open` (priority 1) — fires before Elementor's own header, so ours renders first. A `CGPT_HEADER_RENDERED` constant guard prevents a double render.
- **CSS scope:** Every selector is prefixed `#cgpt-header`. Nothing leaks into the rest of the page.
- **Design tokens:** CSS custom properties defined on `#cgpt-header` (colours, radii, max-width). Extracted from Elementor template 59911 / post-5991.css to match the existing design exactly.
- **Nav menus:** 7 WordPress menus registered (`cgpt_header_main`, `cgpt_header_topbar`, `cgpt_header_platform`, `cgpt_header_solutions`, `cgpt_header_company`, `cgpt_header_resources`, `cgpt_header_actions`). These drive all header content — labels, URLs, and dropdown sections are managed in **Appearance → Menus**, not hardcoded.
- **Dropdown panels:** Mega-menu panels (Platform, Solutions, Company, Resources) are driven by menu sections with structured `children` arrays. Enterprise section within Platform is auto-detected by CSS class `cgpt-enterprise-section` or section title "Enterprise".
- **Transparent/solid scroll state:** Homepage (`body.home`) starts transparent; all other pages start solid-white. `is-scrolled` class toggled at `scrollY > 10`.
- **Wrapper padding offset:** Parent theme JS measured `#elementor-header` for padding-top. We now set `#wrapper` padding-top directly from `header.offsetHeight` in JS, and expose `--cgpt-panel-top` CSS variable for panel positioning.
- **Mobile breakpoint:** `≤1024px` — slide-in panel from right with scrim, hamburger icon toggle, accordion sub-menus.

**What was tested:**
- Desktop: hover-open mega dropdowns, diagonal cursor movement from trigger to panel (no accidental close), Escape key, overlay click-to-close
- Mobile: hamburger open/close, accordion sub-menus, scrim tap-to-close, body scroll lock, resize cleanup
- Scroll state: transparent on homepage at top, solid-white everywhere else and on scroll
- Active state: current page nav item gets `is-current` pill indicator
- Admin bar: z-index layering with WP admin bar
- Elementor stylesheet conflict: `button:hover { color: #fff }` from Elementor Kit neutralised with scoped override so nav trigger labels stay visible
- PHP syntax: `php -l` clean on all changed files

---

## Upcoming phases

### Phase 2 — Footer
Replace the Elementor footer template with custom PHP/CSS.

### Phase 3 — Landing pages
The flexible landing-page system (`page-templates/page-flexible.php` + `inc/blocks.php`) already renders without Elementor. Pages on the flexible template are already Elementor-free. Work here is migrating any remaining Elementor-built pages to the flexible template.

### Phase 4 — Blog / archive templates
`single.php`, `index.php`, `search.php` are child-theme overrides but still pull layout from the parent Starto theme, which uses Elementor for the page frame. These need their own template pass.

### Phase 5 — Remove Elementor plugin
Once all surfaces are migrated and verified on staging and production, deactivate and remove the Elementor plugin.

---

## How to orient yourself (for new contributors)

1. **Read `AGENTS.md`** — workflow rules, deploy pipeline, high-risk files, block maintenance contract.
2. **The header lives in three files:** `templates/template-elementor-header.php` (markup), `assets/css/header.css` (styles), `assets/js/header.js` (interactions). `functions.php` wires them in.
3. **Nav content is managed in Appearance → Menus** in the WP admin — not in the PHP files. If a link is wrong, fix it in the menu, not in the template.
4. **Dev branch is the working branch** for Elementor removal. Changes are tested on staging before merging to `main` → production.
5. **Deploy:** push to `main` → auto-deploys to staging in ~16s. Production deploy is a manual workflow trigger in GitHub Actions.

---

## Deploy pipeline

```
dev  →  (PR or merge)  →  main  →  GitHub Actions  →  staging (auto, ~16s)
                                                    →  production (manual trigger)
```

See `CONTRIBUTING.md` and `AGENTS.md` for full deploy details and SSH access.
