# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A **single-page static website**, not an application. The whole site is one HTML file that was
saved from a WordPress + Elementor site (`thargelion.se`) via the browser's "Save page as".
There is **no build system, no tests, no package manager, and no dependencies** — editing means
changing the HTML and viewing it in a browser (open the `.htm` file directly; no server needed).

The active project is to turn this saved copy into a clean, self-contained static site: removing
interactive/WordPress cruft, deleting unwanted sections, and replacing remote references to the old
`thargelion.se` domain with local assets.

## Project requirements (treat as acceptance criteria)

Every change must keep all three of these true:

1. **Works fully offline — no internet connection.** The page must load and render completely with
   networking disabled. No reference may resolve to a remote host (`thargelion.se`, `youtube.com`,
   `s.w.org`, fonts.googleapis.com, etc.); every asset — CSS, JS, fonts, images, favicons — must be
   local under `Home – Thargelion_files/` or `images/`. Remote references are either localized
   (download into `images/`) or removed. This makes the "de-WordPress-ifying checklist" below a hard
   requirement, not a nicety: group 1 (active remote loads) must reach zero.
2. **No script errors.** The page's JavaScript must run cleanly in the browser console — no errors and
   no failed requests. WordPress/Elementor leftovers that can only work against a live backend
   (`wp.apiFetch`, the REST nonce endpoint, emoji/polyfill loaders, oEmbed) should be removed or
   neutralized rather than left to throw offline. Helper `.ps1` scripts must also run without errors.
3. **English only.** All visible text and language-bearing attributes must be in English. Translate
   leftover Swedish (e.g. "Hoppa till innehåll", "Slå på/av mobilmeny", "Drivs med WordPress",
   "Designad med glädje av") and set the document language to English (`<html lang="en">`).

## Layout

- `Home – Thargelion.htm` — the entire page. **Filename contains an EN DASH (`–`, U+2013), not a hyphen.**
- `Home – Thargelion_files/` — CSS, JS, and images the browser saved locally (also en-dash named).
  Most of the page's stylesheets/scripts are referenced from here.
- `images/` — clean replacement assets added by the site owner. Prefer linking new/repaired image
  references here (relative path `images/...`) rather than to the old domain or the saved `_files` dir.

## Critical gotchas

**The en dash in filenames breaks hardcoded PowerShell paths.** Console encoding corrupts the `–`
character, so `Get-Content -LiteralPath '...Home – Thargelion.htm'` fails with "file not found".
Always resolve the file with a wildcard instead:
```powershell
$path = (Get-ChildItem -LiteralPath 'c:\dev\tharga\Websites\thargelion' -Filter '*Thargelion.htm' | Select-Object -First 1).FullName
```

**The Bash tool mangles PowerShell `$variables`** (they get eaten before PowerShell sees them).
Do **not** pass multi-statement PowerShell via `powershell -Command "..."` through the Bash tool.
Instead, write a temporary `.ps1` script with the Write tool and run it, then delete it:
```
powershell -NoProfile -ExecutionPolicy Bypass -File "c:\dev\tharga\Websites\thargelion\_edit.ps1"
```
The Grep/Glob/Read/Edit tools work fine on the en-dash path directly — prefer them for simple work.

**File encoding:** UTF-8 **without BOM**, **LF** line endings, with some **very long single lines**
(Elementor renders each element as one unbroken line). When rewriting the file from a script,
preserve this: `[System.IO.File]::WriteAllText($path, $text, (New-Object System.Text.UTF8Encoding($false)))`.

## Editing approach

- **Small text/attribute changes:** use the Edit tool directly.
- **Structural removals or repetitive bulk edits:** use a PowerShell script (regex with
  `RegexOptions.Singleline`, or line-range slicing of `$text -split "\`n", -1`). After saving,
  **verify with Grep** and read the cut boundary to confirm nesting is intact, then **delete the temp script**.

**This is Elementor markup.** Every visual element is a `<div class="elementor-element … elementor-widget-X" data-id="…">`
containing a `elementor-widget-container` and deeply nested closing `</div>`s. To remove an element
cleanly, match its full block — e.g. a button widget is
`elementor-widget-button` → `elementor-widget-container` → `elementor-button-wrapper` → `<a>` followed
by exactly three closing `</div>`s. `data-id` values are unique and make good anchors for targeting one element.

## De-WordPress-ifying checklist (project context)

References to the old site fall into groups, in rough priority:
1. **Active remote loads** (favicons, image `srcset`, the `wp-emoji` and `wp-polyfill-*` scripts) — these
   actually fetch from `thargelion.se`. Missing images can be downloaded from the source URLs
   (`https://thargelion.se/wp-content/uploads/...`) straight into `images/` via `Invoke-WebRequest`.
2. **Body links** (skip link, search form `action`, footer logo links, `mailto:`).
3. **`<head>` SEO/metadata** (canonical, OpenGraph, JSON-LD, RSS feeds, REST/oEmbed, shortlink) — dead WP cruft.
4. **Inline JS config** (`wp.apiFetch`, Elementor `urls.assets`).

The site logo links to `/` (root-relative). New image links should be relative (`images/...`) so they
work both when opened as a local file and when served.
