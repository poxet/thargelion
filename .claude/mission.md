# Mission: Thargelion website

## What this is

A **single-page static website**, not an application. The whole site is one HTML file (`index.html`)
that was saved from a WordPress + Elementor site (`thargelion.se`) via the browser's "Save page as".
There is **no build system, no tests, no package manager, and no dependencies**.

The active project is to turn this saved copy into a clean, self-contained static site: removing
interactive/WordPress cruft, deleting unwanted sections, and localizing remote references to the old
`thargelion.se` domain.

> **Applicability note:** this is a static site, not a .NET repo. The `dotnet build/test`, NuGet,
> and most coding/feature-branch rules in `shared-instructions.md` do not apply literally here.
> The session-continuity flow (read this file, check `plan/`, External References) still applies.

## Build & Run

No build or test step. **"Run" = open `index.html` directly in a browser** (no server needed).

## Project requirements (treat as acceptance criteria)

Every change must keep all three true:

1. **Vital parts work offline; online features are allowed.** Opened with no internet connection, the
   *core* experience must fully render and work — layout, styling, content, images, icons, navigation —
   all served from local files (`assets/`, `images/`, `scripts/`). **Online/external features are
   permitted as progressive enhancement** (web fonts from a CDN, external hyperlinks, embedded map/video,
   analytics): they may use the network, but the core page must degrade gracefully when they are
   unavailable — never a broken layout or blocking error. Judge each remote reference by that standard
   rather than deleting it reflexively. One distinction: dead WordPress/Elementor *backend* plumbing
   (`wp.apiFetch`, REST nonce endpoint, oEmbed, emoji loaders pointing at the old site) is **not** an
   online feature — it only ever worked against the old backend and just errors now, so remove/localize it.
2. **No breaking script errors.** The page's JavaScript must run without console errors that break the
   core experience, online or offline. An allowed online feature that cannot reach the network offline
   must fail quietly / degrade gracefully rather than throw. Helper `.ps1` scripts must also run cleanly.
3. **English only.** All visible text and language-bearing attributes must be in English; document
   language is `<html lang="en">`. Remaining Swedish lives only in non-rendered config (api-fetch
   `sv_SE` locale, Elementor i18n strings).

## Layout

- `index.html` — the entire page (one large HTML file). Originally saved as `Home – Thargelion.htm`, since renamed.
- `assets/` — **all in-use** CSS, JS, and images (originally the en-dash-named `Home – Thargelion_files/`,
  since renamed and pruned of dead files; the page's used images — favicons, the `humanoid` set, the
  `thargelion-bkg.jpg` hero background — were moved here too). Referenced as `assets/...` from `index.html`;
  CSS references siblings directly (e.g. `post-203.css` → `thargelion-bkg.jpg`). CSS still uses `../fonts/`
  (resolves to a root `fonts/` that doesn't exist → icon fonts broken), and the Google-Fonts CSS
  (`css`, `css(1..3)`) pulls woff2 from `fonts.gstatic.com` (CDN — degrades to system fonts offline).
- `images/` — **spare/unused** images the site owner added for future use. Nothing here is referenced by
  the page. Leave these in place; the owner wants them available.
- `scripts/` — JS downloaded from the old site to localize it (currently the `wp-polyfill-*` files).
  Reference as relative `scripts/...`.

## Working notes / gotchas

- **The Bash tool mangles PowerShell `$variables`** (they get eaten before PowerShell sees them). Do not
  pass multi-statement PowerShell via `powershell -Command "..."` through Bash. Instead write a temporary
  `.ps1` with the Write tool, run it with `powershell -NoProfile -ExecutionPolicy Bypass -File "...ps1"`,
  then delete it. Grep/Glob/Read/Edit work fine directly — prefer them for simple work.
- **File encoding:** UTF-8 **without BOM**, **LF** line endings, with some **very long single lines**
  (Elementor renders each element as one unbroken line). When rewriting from a script, preserve this:
  `[System.IO.File]::WriteAllText($path, $text, (New-Object System.Text.UTF8Encoding($false)))`.
- **Never mix logging and return values in a PowerShell function.** Everything emitted to the output
  stream becomes the return value, so a function that `Write-Output`s progress *and* returns text returns
  an array `[log…, text]`; capturing that and writing it back corrupts the file (log lines get prepended).
  Inline the `.Replace()`s, or send logs to `Write-Host`. Sanity-check the file start after any scripted rewrite.
- **This is Elementor markup.** Every visual element is `<div class="elementor-element … elementor-widget-X"
  data-id="…">` containing a `elementor-widget-container` and deeply nested closing `</div>`s. To remove an
  element cleanly, match its full block; `data-id` values are unique and make good anchors.
- **Scan the saved CSS too, not just `index.html`.** Background images live in `assets/post-203.css` as
  `url(...)` rules (Elementor per-page CSS) — that's where the hero background hid. Grep `assets/*.css` for
  `url\(\s*['"]?https?://` and `thargelion.se`. A `url()` only loads if an element matches its selector,
  so a rule for a deleted section is harmless dead CSS.
- **En dash (historical):** the assets folder used to be `Home – Thargelion_files/` (en dash `–`/U+2013),
  which PowerShell's console encoding corrupted. Both the main file and folder now have plain names. If a
  stray en-dash path ever recurs, build the literal with `[char]0x2013` or resolve via a wildcard filter.

## De-WordPress-ifying status

Judge each old-site reference against requirement #1 (does the *core* still work offline?). Groups:

1. **Active remote loads** (favicons, image `srcset`, `wp-emoji`/`wp-polyfill-*` scripts, CSS `url()`
   backgrounds) — *done.* Favicons + `humanoid` + hero `thargelion-bkg.jpg` → `assets/`; polyfills →
   `scripts/`; emoji loader, `wp.apiFetch`, Elementor `urls.assets` localized/neutralized; YouTube video
   widget (+ `Layer-9` thumbnail) and dead `b1.png` background removed.
2. **Body links** — *done.* Header search + its form `action` removed; skip link → `#content`; footer logo → `/`.
3. **`<head>` SEO/metadata** (canonical, OpenGraph, JSON-LD, RSS feeds, REST/oEmbed, shortlink, generator,
   `gmpg`/Yoast, `dns-prefetch` hints) — *done*, all removed.
4. **Inline JS config** (`wp.apiFetch`, Elementor `urls.assets`) — *done.*

**Current state:** the only remaining external reference is the footer "Designed with joy by Rancio"
credit link (`www.rancio.com`), kept intentionally (allowed online feature). The site logo links to `/`
(root-relative); new image links should be relative so they work both as a local file and when served.

## External References

- **Shared instructions**: `$DOC_ROOT/Tharga/shared-instructions.md`
- **Plan directory**: `$DOC_ROOT/Tharga/plans/thargelion` — future features under `planned/`
  (numbered `01-…`, `02-…`), completed under `done/`.
