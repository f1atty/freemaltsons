# Freemaltsons — Claude Instructions

## Critical: Two versions of the app exist

- `index.html` (root) — PRODUCTION. GitHub Pages. This is the only file to edit.
- `static/index.html` — local dev only (Flask). Do NOT edit unless explicitly asked.

Always edit the root `index.html`. Never edit `static/index.html` unless the user says so.

## Git workflow

Remote is often ahead of local (GitHub API writes from the app). Always use this pattern:

```
git stash && git pull --rebase origin main && git stash pop && git push origin main
```

Commit and push after every change — do not batch multiple features into one commit.

## Context management

Proactively warn the user to run `/compact` or start a new session WELL BEFORE context runs out — do not wait until it is nearly full. Suggest it early, after a reasonable number of exchanges or when the conversation has covered several topics. Each new session loads CLAUDE.md and memory files automatically so no context is lost.
