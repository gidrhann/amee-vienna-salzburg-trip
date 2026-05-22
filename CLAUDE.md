# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a single-file static website (`index.html`) deployed to GitHub Pages at:
**https://gidrhann.github.io/amee-vienna-salzburg-trip/**

GitHub repo: `gidrhann/amee-vienna-salzburg-trip` (public)

The page is a 2026 Austria travel itinerary (Vienna + Salzburg, AMEE medical education conference) with inline editing, password protection, sidebar toggle, and GitHub push capability built into the browser.

## Git Workflow — Critical

The user edits content directly in the browser and pushes via the in-page GitHub API button. **Always `git pull origin master` before making any local changes**, then push normally — never `git push --force`.

```bash
git pull origin master   # fetch user's browser edits first
# make changes
git add index.html
git commit -m "..."
git push                 # no force needed
```

If push is rejected, `git pull --rebase` and resolve conflicts manually — keep user's content edits, layer structural/feature changes on top.

### Common Pitfall: editBtn hidden in source HTML

If the user pushes from the browser while in edit mode, `#editBtn` will be committed with `style="display: none;"`. Fix by removing that inline style from the HTML directly.

## Architecture — index.html

Everything lives in one file. The sections in order:

1. **CSS** — design tokens via `:root` CSS variables (`--green`, `--gold`, `--blue`, `--rose`), responsive grid layout, print styles, lock overlay styles, edit-mode styles, sidebar toggle styles
2. **HTML** — lock overlay → `.layout` (CSS grid: sidebar | main)
3. **JavaScript** (bottom `<script>`) — seven independent systems:

| System | Key identifiers |
|--------|----------------|
| Password auth | `PASS_HASH`, `AUTH_KEY`, `initAuth()`, `submitPassword()` |
| Sidebar toggle | `SIDEBAR_KEY`, `toggleSidebar()`, `initSidebar()`, `.sidebar-hidden` |
| Print | `printPage()`, `#printToast` |
| Tab filter | `handleTabClick()`, `.tab-btn[data-filter]`, `.day[data-tags]` |
| Edit mode | `EDITABLE_SEL[]`, `enterEdit()`, `exitEdit()`, `setEditables()` |
| localStorage persistence | `STORAGE_KEY = 'amee_trip_v1'`, `loadSaved()`, `clearStorage()` |
| GitHub push | `GH_OWNER/GH_REPO/GH_FILE`, `TOKEN_KEY`, `pushToGitHub()` |

All button events use **document-level event delegation** (single `document.addEventListener('click', ...)`) — this is intentional so events survive after `loadSaved()` replaces `.layout`'s innerHTML.

## localStorage Keys

| Key | Purpose |
|-----|---------|
| `amee_auth_v1` | SHA-256 hash of entered password (persists auth across sessions) |
| `amee_trip_v1` | Saved `.layout` innerHTML (user's content edits) |
| `amee_sidebar_v1` | Sidebar collapsed state (`'1'` = hidden) |
| `gh_pat_amee` | GitHub Personal Access Token for in-browser push |

## Password

SHA-256 hash of the login password is hardcoded as `PASS_HASH` in the script. To change the password, compute `sha256(newPassword)` and replace the constant. Current hash corresponds to the password set by the owner.

## loadSaved() — Important Behaviour

`ORIGINAL_LAYOUT_HTML` is captured at script start (before `loadSaved()` runs) to enable reload-free restoration.

After restoring `.layout` innerHTML from localStorage, the code **must** reset edit-mode state:
- Remove all `[contenteditable]` attributes
- Show `#editBtn` via `removeAttribute('style')`
- Remove `body.edit-mode` class

Failing to do this causes the edit button to remain hidden after a page refresh.

## clearStorage() — No Reload

`clearStorage()` does **not** call `location.reload()`. Instead it:
1. Removes `STORAGE_KEY` from localStorage
2. Restores `ORIGINAL_LAYOUT_HTML` directly into `.layout`
3. Manually resets all edit-mode state

This avoids browser bfcache issues where `location.reload()` could restore a stale JS state with `#editBtn` still hidden.

## Auto-save — Clean State

The auto-save debounce (800 ms) temporarily strips `contenteditable` attributes and restores `#editBtn` visibility before writing to localStorage, then restores the live edit state. This ensures localStorage always contains a clean (non-edit-mode) snapshot.

## pushToGitHub() — Clean State

Before serialising `document.documentElement.outerHTML`, the push function:
1. Temporarily removes all `contenteditable` attributes
2. Removes `body.edit-mode` and hides `#editBar`
3. Captures the HTML
4. Restores edit state

This ensures the committed HTML never contains edit-mode artefacts (`display:none` on `#editBtn`, stray `contenteditable` attributes).

## GitHub Pages Deployment

- Branch: `master`, path: `/` (root)
- File served: `index.html`
- No build step — changes go live ~1 minute after push
