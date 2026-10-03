# L'inizio Pasta Bar — Website Project Context

Paste or keep this file at the repo root as `CLAUDE.md` so Claude Code has full context before making changes.

---

## What this is

A custom website for **L'inizio**, an Italian pasta bar in Leuven, Belgium. Single-page bilingual (NL/EN) site with an online ordering flow that sends orders via WhatsApp, plus a CMS so the non-technical owner can edit the menu, prices, hours, and notices from his phone.

- **Developer:** Sonu Kumar Suman
- **Owner (client, non-technical):** Salvatore Cascella — L'inizio Pasta-bar, Parijsstraat 39, 3000 Leuven
- **Live site:** https://linizio.be  (also https://www.linizio.be)
- **Admin/CMS:** the owner currently opens `https://darling-pithivier-818db9.netlify.app/admin` — see CMS section, this matters

---

## Architecture (read this before touching anything)

The setup spans three services. Understanding the split prevents most mistakes.

| Service | Role | Notes |
|---|---|---|
| **GitHub** (`sonuyou19-max/linizio-leuven`) | Source of truth — holds all files | Sonu is a collaborator; owner is account owner |
| **Cloudflare Pages** | Hosts the live site at linizio.be | Watches GitHub `main`, auto-deploys on push. Free plan = 500 builds/month |
| **Netlify** (`darling-pithivier-818db9`) | CMS login ONLY — Netlify Identity + Git Gateway | Hosts nothing real. GitHub auto-deploy is DISCONNECTED to save credits |

**Content flow when the owner edits in the CMS:**
```
Owner edits in Decap CMS  →  Git Gateway writes content/*.json to GitHub  →  Cloudflare detects push, rebuilds  →  linizio.be updates (~30s)
```

**Two copies of the admin page exist** (this has caused repeated confusion):
- GitHub copy → served at `linizio.be/admin` (updates on git push)
- Netlify copy → served at `darling-pithivier-818db9.netlify.app/admin` (deployed by MANUAL DRAG-AND-DROP only; a git push does NOT update it)

The owner uses the Netlify URL. So a fix to `admin/index.html` pushed to GitHub does NOT reach him until the same file is also drag-dropped onto the Netlify site. Long-term goal: move him to `linizio.be/admin` and retire the Netlify copy, keeping Netlify only for Identity + Git Gateway (which are API endpoints, not files).

---

## Files

| Path | What it is | Who edits it |
|---|---|---|
| `index.html` | The entire website (HTML + CSS + JS inline, ~199KB) | Sonu / Claude Code |
| `admin/index.html` | Decap CMS loader page | Sonu / Claude Code |
| `admin/config.yml` | CMS field definitions | Sonu / Claude Code |
| `content/general.json` | phone1, phone2, address | **OWNER via CMS — never overwrite** |
| `content/hours.json` | opening hours + special_notice_nl/en | **OWNER via CMS — never overwrite** |
| `content/menu_suggesties.json` | Suggesties dishes + note_nl/en | **OWNER via CMS — never overwrite** |
| `content/menu_vegetarisch.json` | Vegetarian dishes | **OWNER via CMS — never overwrite** |
| `content/menu_vlees.json` | Meat dishes | **OWNER via CMS — never overwrite** |
| `content/menu_vis.json` | Fish dishes | **OWNER via CMS — never overwrite** |
| `content/menu_extra.json` | Supplements | **OWNER via CMS — never overwrite** |

### ⚠️ The single most important rule
**NEVER push `content/*.json` to GitHub.** The owner manages those through the CMS. Overwriting them wipes his dishes, prices, KV text, and notices. When deploying code changes, upload ONLY `index.html`, `admin/index.html`, and `admin/config.yml`.

---

## config.yml backend (top of admin/config.yml)
```yaml
backend:
  name: git-gateway
  branch: main
  base_url: https://darling-pithivier-818db9.netlify.app
```
`base_url` tells Decap where Netlify Identity lives (login is on Netlify, site is on Cloudflare — different domains, so it must be explicit).

---

## admin/index.html — the Decap version pin (CRITICAL, recurring issue)

The CMS loads Decap from a CDN. The script tag **must pin an exact version**:
```html
<script src="https://unpkg.com/decap-cms@3.12.2/dist/decap-cms.js"></script>
```
Do NOT use `@^3.0.0` or `@latest`. A floating range makes unpkg serve whatever the newest 3.x build is at page-load time. This caused TWO outages:
- 3.13.0 → image-preview regression
- 3.15.0 (published 2026-07-23) → broke the owner's login entirely for ~23h until maintainers hot-fixed with 3.15.1

Pinning to **3.12.2** (the build that ran fine for two months) freezes it so an upstream release can never break his admin again. Trade-off: no future Decap fixes — acceptable for a menu editor behind invite-only login; revisit yearly.

**Permanent option (not yet done):** self-host the file. Commit `decap-cms.js` (5.5MB) + its `373.decap-cms.js` chunk into `admin/` and load `./decap-cms.js`. The bundle has no hardcoded CDN path and uses `document.currentScript`, so same-folder hosting works. This removes the unpkg dependency entirely.

---

## Website features (all implemented, in index.html)

- Custom canvas hero animation, mobile-optimised
- Bilingual NL/EN — 150+ translated elements via a `TRANSLATIONS` object + `setLang(lang)` function
- Menu: 4 tabs (Suggesties, Vegetarisch, Vlees, Vis) + Supplementen, ~75 dishes, built at runtime by `buildTab()` from the JSON
- Cart system → WhatsApp order message, pickup-time selector, receipt modal, "Toon aan raam" chef screen
- KV (koppelverkoop/bundle) ordering — dual buttons `+` and `+ KV`; KV label shown as "KV {dish name}" (prefix, not suffix); owner-added KV descriptions (e.g. "medium bekers") wrap below price, buttons stay top-right aligned
- Fly-to-cart animation on add
- Sold-out toggle per dish via CMS (fades item, shows badge, disables buttons)
- Order window: WhatsApp orders only 09:30–10:30 & 13:30–16:30 **Belgium time**, hard-blocked outside with phone numbers shown
- Special closure notice (CMS-driven, bilingual) shown in TWO places: top info strip + above hours table; hidden when empty
- Suggesties note (CMS-driven, bilingual, hidden when empty)
- Dishes display in **CMS drag-and-drop order** (owner reorders in admin; automatic price-sort was REMOVED because it overrode his manual order)
- Security headers: CSP meta tag, X-Content-Type-Options, Referrer-Policy

---

## Hard-won gotchas (things that broke before — don't repeat)

1. **Timezone check must use `Intl.DateTimeFormat('en',{timeZone:'Europe/Brussels',...}).formatToParts()`.** The old `new Date(d.toLocaleString('en-GB',{timeZone}))` returns Invalid Date on iOS Safari → NaN → blocks every order. This was a real outage.
2. **Order-window boundaries are inclusive** (`>= 9:30 && <= 10:30`). Strict `<` blocked orders at exactly 10:30/16:30.
3. **Language toggle must read CMS data, not hardcoded TRANSLATIONS, for CMS-driven text** (special notice, suggesties note). Store fetched hours/suggesties JSON on `window._hoursData` / `window._suggestiesData`; `setLang` reads those. Also: the generic `[data-tx]` loop uses `el.children.length===0` so it only touches leaf nodes — never wipe a parent's nested HTML by setting its innerHTML.
4. **`e.textContent = ...` on an `<a>` wipes child elements** (icons). When CMS updates phone numbers, target the inner `.contact-val` span, not the whole anchor.
5. **Contact icons are inline SVG**, not emoji (emoji rendered tiny/inconsistent).
6. **CMS "changes published but not showing"** has had two causes historically: (a) expired Netlify Identity token → `Gotrue-js: failed getting jwt access token` → fix by logout/login or password reset in Netlify → Identity; (b) stale browser cache on one device → clear Safari website data for `netlify.app`. If it's one device only, it's cache/token, not code.
7. **DNS/domain** is managed at registrar **dulouca.be**. Nameservers point to Cloudflare (`dalary.ns.cloudflare.com`, `joaquin.ns.cloudflare.com`). MX record (`mail.linizio.be`) and SPF TXT must NEVER be touched — that's the owner's email.

---

## Deploy checklist (for a code change)

1. Edit `index.html` (and/or `admin/config.yml`) locally.
2. Validate: no JS syntax errors (run each `<script>` through `node --check`).
3. Push ONLY the changed non-content files to GitHub `main`.
4. Cloudflare auto-rebuilds (~30s). Verify on linizio.be (hard refresh / incognito).
5. If `admin/index.html` changed, ALSO drag-drop the `admin/` folder (with `index.html` + `config.yml`) onto the Netlify site, since the owner uses the Netlify admin URL.
6. Never touch `content/*.json`.

---

## Commercial context (for reference, not code)

- Original build: €650, paid, handover done first week of May 2026 (accounts created fresh in owner's name, documents signed).
- Post-handover enhancements (suggesties note, closure notice, dish reordering, KV layout): quoted €125.
- WhatsApp timezone bug fix: free (it was a defect).
- Rate for future major work: quoted per request.
- Owner wants to meet to discuss further enhancements; Sonu is scheduling for ~2 weeks out, in Leuven.

---

## Open / possible next work

- Move owner to `linizio.be/admin` and retire the Netlify admin copy (keep Netlify for Identity + Git Gateway only) — removes the two-copies drift problem permanently.
- Self-host `decap-cms.js` to eliminate the unpkg dependency.
- Enhancements Salvatore wants to discuss (TBD at the meeting).
