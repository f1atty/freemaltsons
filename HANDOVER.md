# Freemaltsons — Handover

**Owner:** Daniel Fainsinger (Strategy & Ops)
**Last updated:** 2026-07-02
**Status:** Active — live on GitHub Pages
**Repo:** https://github.com/f1atty/freemaltsons.git (branch: `main`)

## What it is

A single-page web app for tracking a whisky club's tasting nights ("Freemaltson's Whisky Nights"). It records each session (host, whisky, region, RRP, image, per-member ratings and notes), and surfaces a session grid, a ratings leaderboard, a spend/hosting ledger, and a map of distillery origins. It is a personal project for the 7-member club, installable as a PWA.

## Current status

- Live and in use on GitHub Pages at https://f1atty.github.io/freemaltsons/
- Repo transferred from `danielf-neara` to `f1atty` on 2026-07-02; all in-code URLs and the local remote point to `f1atty`
- Data writes from the app work via a fine-grained GitHub PAT (Contents: Read/write, scoped to `freemaltsons`) — confirmed working after the transfer
- Core features complete: session grid + filters, ratings leaderboard, ledger, map, upcoming sessions, calendar export, per-member PIN sign-in
- Local `main` is clean and in sync with `origin/main`

## How to run / access

### Production (the one that matters)
- **No build step, no server.** Root `index.html` is served statically by GitHub Pages.
- Open https://f1atty.github.io/freemaltsons/ (or install as a PWA).
- **Sign-in:** each member enters their PIN (hashed, defined in `MEMBER_PINS` in `index.html`).
- **Data reads:** `data/sessions.json` via the GitHub Contents API; `data/whisky-library.json` via a raw.githubusercontent URL.
- **Data writes:** require a GitHub PAT saved in the app's Settings (stored in `localStorage` under `freemaltsons_pat`). The token must be fine-grained, Contents: Read and write, scoped to `f1atty/freemaltsons`.
- **Bottle image search (optional):** a Google Custom Search API key + CX, entered in Settings (`freemaltsons_google_key`, `freemaltsons_google_cx`).

### Local dev (Flask — rarely needed)
```
./run.sh          # installs flask/requests/bs4, serves http://localhost:5001
```
- Serves `static/index.html` and reads/writes the local `data/*.json` files via `app.py`.
- This path is for local experimentation only and is not what users hit.

## How it works

| Concern | Production (`index.html`) | Local dev (`app.py` + `static/index.html`) |
|---------|---------------------------|---------------------------------------------|
| Hosting | GitHub Pages (static) | Flask, port 5001 |
| Read data | GitHub Contents API + raw URL | Local filesystem |
| Write data | GitHub Contents API (PUT, needs PAT) | Local filesystem |
| Auth | Per-member PIN (client-side hash) | Same |
| Secrets | PAT + Google keys in `localStorage` | n/a |

Key config constants live near the top of the app script in `index.html`:
- `GITHUB_REPO = 'f1atty/freemaltsons'`, `GITHUB_DATA = 'data/sessions.json'`
- `githubLoad()` reads sessions (captures the file SHA), `githubSave()` PUTs updated sessions with that SHA.

## File / directory map

| Path | What it is |
|------|-----------|
| `index.html` | **PRODUCTION app.** The only file to edit for user-facing changes. |
| `static/index.html` | Local-dev copy served by Flask. Do NOT edit unless explicitly asked. |
| `app.py` | Flask backend for local dev (port 5001). |
| `run.sh` | Starts the local Flask app and opens the browser. |
| `requirements.txt` | Local-dev Python deps (flask, requests, beautifulsoup4, openpyxl). |
| `data/sessions.json` | Session records + top-level `members` array. Large (base64 images embedded). |
| `data/whisky-library.json` | Reference whisky library. |
| `manifest.json` | PWA manifest. |
| `icons/` | PWA icons. |
| `Freemaltsons crest.png` | App crest/logo. |
| `Freemaltson's Whisky.xlsx` | ~12 MB source workbook, committed via a `.gitignore` exception. |
| `docs/superpowers/plans/` , `docs/superpowers/specs/` | Upcoming-sessions plan + design spec (2026-03-12). |
| `CLAUDE.md` | Project instructions for Claude. |

## Key decisions & gotchas

- **Two versions exist.** Always edit root `index.html`. `static/index.html` and `app.py` are local-dev only and must not be edited unless asked.
- **Remote is often ahead of local.** The app itself commits to `main` via the GitHub API (author `f1atty`, message "Update sessions"). Always pull before working: `git stash && git pull --rebase origin main && git stash pop && git push origin main`. Commit and push after every change; do not batch.
- **Repo transfer, 2026-07-02.** Moved `danielf-neara/freemaltsons` → `f1atty/freemaltsons` via GitHub transfer (accepted by f1atty). Old URLs redirect. In-code URLs, the token hint, and the workspace `CLAUDE.md` map were updated. Pages URL changed to `f1atty.github.io/freemaltsons`.
- **PAT ownership.** The write token must belong to (or have access to) `f1atty/freemaltsons`. A wrong or expired PAT shows "Sync failed" / "No token" in the sync status.
- **Host name aliases.** Member/host names have been entered inconsistently; `HOST_ALIASES` (in both `app.py` and `index.html`) normalises them (e.g. `brass`→`Braas`, `willie`→`Joess`).
- **7 fixed members** stored in `data.members`.
- **Geocode cache** lives in `localStorage` under `fm_geocache`.

## Open tasks / next steps

- [ ] No open tasks tracked. Migration to `f1atty` is complete and verified.
- [ ] (Optional) Decide whether to keep the local Flask dev path or retire it, since production is fully static.

## Dependencies, integrations & contacts

- **GitHub Pages** — hosting for production.
- **GitHub Contents API** — data read/write; requires fine-grained PAT (Contents R/W, scoped to `f1atty/freemaltsons`).
- **Google Custom Search API** — optional bottle image lookup (key + CX in Settings).
- **Wikipedia REST API** — distillery/whisky summaries in map popups.
- **Leaflet / OpenStreetMap** — map rendering and geocoding.
- **Contact:** Daniel Fainsinger (owner). GitHub account `f1atty` owns the repo.
