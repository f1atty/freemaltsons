# Upcoming Sessions Feature — Design Spec
**Date:** 2026-03-12
**Status:** Approved

## Overview

Allow a host to schedule a future whisky night before the session occurs. The whisky can be hidden from other members (or left unset) until the host reveals it on the night. Sessions also gain a location field and calendar export buttons.

---

## Data Model

Two new optional fields added to each session object in `sessions.json`:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `location` | string | `null` | Venue or address for the session |
| `whisky_hidden` | boolean | `false` | If true, whisky name is hidden from non-host members |

`whisky` becomes optional — it may be `null` or empty string when the host hasn't decided yet.

**Backwards compatibility:** Any session where `whisky_hidden` is absent (`undefined`) must be treated as `false`. Never assume the field is present.

**Valid state:** `whisky: null` combined with `whisky_hidden: true` is explicitly permitted. It means the host intends to add and hide the whisky before the night.

**Example:**
```json
{
  "id": "X:I",
  "date": "2026-04-15",
  "host": "Flatty",
  "whisky": "Lagavulin 16",
  "whisky_hidden": true,
  "location": "Flatty's place",
  ...
}
```

---

## "Today" Boundary

Anywhere the spec uses `date >= today` or `date < today`, "today" is computed using the user's **local date**:
```js
const today = new Date().toLocaleDateString('en-CA'); // yields "YYYY-MM-DD" in local time
```
Do not use `new Date().toISOString().slice(0,10)` (UTC-based, incorrect for Australian users late at night).

`openModal()` currently defaults the date input using `new Date().toISOString().split('T')[0]` — this must also be updated to use the local date pattern above.

---

## Session Filters

The existing `loadData()` filter:
```js
allSessions = (data.sessions || []).filter(s => s.whisky || s.host);
```
An upcoming session with `whisky: null` passes this filter because `s.host` is truthy — this is correct and must not be tightened.

**`renderHeaderStats` and `renderLedger`:** Upcoming sessions (where `date >= today`) are **excluded** from header stats and ledger counts. They have not yet occurred, so they should not inflate session counts or hosting tallies. The existing `s.whisky` filter in these functions incidentally achieves this for null-whisky sessions, but the correct intent is to filter by `date < today` (past sessions only).

**Search/filter grid:** An upcoming session with `whisky: null` appears in the grid when no search text is entered. It will not match a whisky-text search, which is correct — it has no whisky to match against. Do not "fix" this behaviour. The `location` field is not added to search matching at this stage.

---

## Session ID Sequencing

`computeNextId()` reads the last session in `allSessions` to generate the next ID. If an upcoming session already exists (e.g. `X:I`), a second "Add Session" will generate `X:II`. This is acceptable. No change to `computeNextId()` is required.

---

## Host Identity

The signed-in member is stored in `state.currentMember` (set at sign-in). All host-visibility logic must compare `state.currentMember.toLowerCase()` to `session.host.toLowerCase()`.

At **wizard time** (before a session object exists), comparisons use `newSession.host` (the host value selected in Step 1 of the wizard), not a persisted session object.

---

## Add Session Wizard Changes

### Step 1 — Date / Host
- Add an optional **Location** text input below the existing fields.

### Step 2 — Whisky Selection
- Add an **"Add later"** selectable option at the top of the whisky search step. When selected, bypasses the search and sets `whisky = null` on the session.
- Add a **"Hide whisky until the night"** checkbox. Shown only when `state.currentMember.toLowerCase() === newSession.host.toLowerCase()` (i.e. the signed-in member is adding a session they will host). Sets `whisky_hidden = true` on save.
- The "Add later" and "Hide whisky" options are independently selectable. A session with `whisky: null` and `whisky_hidden: true` is valid (see Data Model).

---

## Card / Grid Display

### Whisky visibility rules
- If `whisky_hidden: true` OR `whisky` is null/empty: show **"???"** in place of the whisky name.
- Exception: if `state.currentMember.toLowerCase() === session.host.toLowerCase()`, show the actual whisky name, or **"Not chosen yet"** if `whisky` is null.

### Location
- Shown below the host name as a small secondary line on the card, only if `location` is non-empty.

### Upcoming session ordering
- Sessions where `date >= today` are considered upcoming (use local date as defined above).
- Upcoming sessions float to the **top of the grid**, above past sessions, with a subtle "Upcoming" label or border accent.

### Ratings tab
- The Ratings tab is hidden entirely for upcoming sessions (`date >= today`). There is nothing to rate yet.

---

## Detail Modal Changes

- Show **location** below the host field if present.
- The Ratings tab is hidden for upcoming sessions.

### Reveal / Add whisky buttons (host only — only shown when `state.currentMember.toLowerCase() === session.host.toLowerCase()`)

| State | Button shown |
|-------|-------------|
| `whisky` set, `whisky_hidden: true` | "Reveal whisky" — sets `whisky_hidden = false` and saves |
| `whisky` null, `whisky_hidden: false` | "Add whisky" — compact inline search within the modal |
| `whisky` null, `whisky_hidden: true` | "Add whisky" only; Reveal is disabled until whisky is set |

The inline "Add whisky" search is a compact input + results list rendered inside the detail modal (no separate modal). Once a whisky is selected, it updates the session and saves.

---

## Edit Modal Changes

The Edit button is only shown when `state.currentMember.toLowerCase() === session.host.toLowerCase()`. The existing code does not gate this — it must be updated as part of this feature.

The Edit modal gains:
- **Location** field (pre-filled from `session.location`).
- **Whisky** field becomes optional (can be cleared to set `whisky = null`).
- **"Hide whisky"** toggle (pre-filled from `session.whisky_hidden`).

---

## Calendar Buttons

Shown on session cards and in the detail modal for any session where `date >= today` (local date).

Two options presented side by side:
- **Add to Google Calendar** — opens a pre-filled Google Calendar URL in a new tab.
- **Download .ics** — generates and downloads an `.ics` file for Apple Calendar / Outlook.

**Card button handlers** must reference the session via `allSessions.indexOf(s)` (consistent with the existing card click handler pattern), not a loop index from `filteredSessions`.

**Event details:**
- Title: `Freemaltsons [session ID] @ [host]` — e.g. `Freemaltsons IX:I @ Flatty`
- Date: session date (all-day event)
- Location: session location if set
- Description: whisky name if `whisky` is set AND (`whisky_hidden` is false OR `state.currentMember === session.host`); otherwise `"Whisky TBA"`.

---

## Reveal Flow

1. Host signs in on the night.
2. Opens the session detail modal.
3. If whisky is set: sees "Reveal whisky" button. Clicks it — `whisky_hidden` set to `false` and saved. All members now see the whisky name.
4. If whisky is not set: sees "Add whisky" inline search. Adds the whisky, then the "Reveal whisky" button appears and can be clicked.

---

## Out of Scope

- Push notifications or email reminders.
- RSVP / attendance tracking.
- Multiple calendar integrations beyond Google Calendar + .ics.
