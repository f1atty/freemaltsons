# Upcoming Sessions Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Allow a host to schedule a future whisky night with a hidden whisky, plus add location and calendar export to all upcoming sessions.

**Architecture:** All changes live in a single file (`index.html`). The app is vanilla HTML/CSS/JS with no build step. Data is stored in `sessions.json` on GitHub and written back via the GitHub Contents API. No test framework exists — verification is done by loading `index.html` directly in a browser (not `static/index.html`). The signed-in member is stored in `sessionStorage` under key `fm_user` (set at line 3616 of `index.html`).

**Tech Stack:** Vanilla JS, CSS custom properties, Leaflet (map only), GitHub Contents API, Web Crypto API (SHA-256 for PINs)

**Spec:** `docs/superpowers/specs/2026-03-12-upcoming-sessions-design.md`

---

## Auth Note (read before implementing)

The signed-in member is stored in `sessionStorage.getItem('fm_user')` — there is no `state` object. Every host check throughout this plan uses a helper function:

```js
function currentMember() { return sessionStorage.getItem('fm_user') || ''; }
```

This is added in Task 1. Do not use `state.currentMember` anywhere.

---

## Chunk 1: Helpers + Stats/Ledger Filter Fix

### Task 1: Add helpers and fix date defaults

**Files:**
- Modify: `index.html` — Helpers section (~line 2376)
- Modify: `index.html:2662` — `openModal` date default
- Modify: `index.html:2424` — `renderHeaderStats` filter
- Modify: `index.html:2449` — `renderLedger` filter

- [ ] **Step 1: Add `localToday()`, `isUpcoming()`, and `currentMember()` helpers**

In `index.html`, find the Helpers section (around line 2376, just after the `esc()` function). Add these three functions:

```js
function localToday() {
  return new Date().toLocaleDateString('en-CA'); // YYYY-MM-DD in local time
}

function isUpcoming(s) {
  return !!(s.date && s.date >= localToday());
}

function currentMember() {
  return sessionStorage.getItem('fm_user') || '';
}
```

- [ ] **Step 2: Fix date default in `openModal()`**

Find line ~2662 inside `openModal()`:
```js
const today  = new Date().toISOString().split('T')[0];
```
Replace with:
```js
const today  = localToday();
```

- [ ] **Step 3: Fix `renderHeaderStats` to exclude upcoming sessions**

Find line ~2424:
```js
const sessions  = allSessions.filter(s => s.whisky);
```
Replace with:
```js
const sessions  = allSessions.filter(s => s.whisky && !isUpcoming(s));
```

- [ ] **Step 4: Fix `renderLedger` to exclude upcoming sessions**

Find line ~2449:
```js
const sessions  = allSessions.filter(s => s.whisky && s.host);
```
Replace with:
```js
const sessions  = allSessions.filter(s => s.whisky && s.host && !isUpcoming(s));
```

- [ ] **Step 5: Manual verification**

Open `index.html` in a browser, sign in. Check that header stats (sessions count, rounds, spend, countries) and ledger tab still show correct numbers for past sessions.

- [ ] **Step 6: Commit**

```bash
cd /Users/danielfainsinger/Documents/GitHub/experiments
git add freemaltsons/index.html
git commit -m "feat(freemaltsons): add localToday/isUpcoming/currentMember helpers; fix stats and ledger to exclude upcoming sessions"
```

---

### Task 2: Add `location` and `whisky_hidden` to session state

**Files:**
- Modify: `index.html:2301` — `newSession` initialiser
- Modify: `index.html:2659` — `openModal` reset
- Modify: `index.html:2834` — `submitSession` session object

- [ ] **Step 1: Update `newSession` initialiser**

Find line ~2301:
```js
let newSession = { host:null, whisky:null, region:null, rrp:null, image_url:null, dm_url:null };
```
Replace with:
```js
let newSession = { host:null, whisky:null, region:null, rrp:null, image_url:null, dm_url:null, location:null, whisky_hidden:false };
```

- [ ] **Step 2: Reset new fields in `openModal()`**

Find line ~2659 in `openModal()`:
```js
newSession = { host: null, whisky: null, region: null, rrp: null, image_url: null, dm_url: null };
```
Replace with:
```js
newSession = { host: null, whisky: null, region: null, rrp: null, image_url: null, dm_url: null, location: null, whisky_hidden: false };
```

- [ ] **Step 3: Include new fields in `submitSession()`**

Find the session object literal inside `submitSession()` (around line 2834). Replace the entire object with:

```js
  const session = {
    id:            document.getElementById('sessionIdInput').value.trim() || null,
    date:          document.getElementById('dateInput').value || null,
    host:          normaliseHost(newSession.host),
    whisky:        document.getElementById('whiskyNameInput').value.trim() || newSession.whisky || null,
    region:        document.getElementById('regionInput').value.trim() || null,
    rrp:           parseFloat(document.getElementById('rrpInput').value) || null,
    image_url:     document.getElementById('imageUrlInput').value.trim() || newSession.image_url || null,
    dm_url:        newSession.dm_url || null,
    location:      document.getElementById('locationInput').value.trim() || null,
    whisky_hidden: newSession.whisky_hidden || false,
    notes:         null,
  };
```

- [ ] **Step 4: Commit**

```bash
cd /Users/danielfainsinger/Documents/GitHub/experiments
git add freemaltsons/index.html
git commit -m "feat(freemaltsons): add location and whisky_hidden to session state and submit"
```

---

## Chunk 2: Add Session Wizard

### Task 3: Add Location field to wizard Step 0

**Files:**
- Modify: `index.html:2057` — Step 0 HTML block

- [ ] **Step 1: Add location input to Step 0 HTML**

Find the Step 0 block (around line 2057). It ends with:
```html
      <div class="host-grid" id="hostGrid"></div>
      <div class="modal-actions">
        <button class="btn-primary" id="hostNextBtn" disabled>Continue</button>
      </div>
    </div>
```

Replace with:
```html
      <div class="host-grid" id="hostGrid"></div>
      <div class="field" style="margin-top:20px">
        <label class="field-label">Location <span class="field-note">optional</span></label>
        <input class="modal-input" id="locationInput" type="text" placeholder="e.g. Flatty's place">
      </div>
      <div class="modal-actions">
        <button class="btn-primary" id="hostNextBtn" disabled>Continue</button>
      </div>
    </div>
```

- [ ] **Step 2: Reset location input in `openModal()`**

After Step 1 is complete, find the block in `openModal()` that resets inputs (around line 2675, e.g. near `document.getElementById('whiskySearchInput').value = ''`). Add:
```js
document.getElementById('locationInput').value = '';
```

- [ ] **Step 3: Manual verification**

Open `index.html`, sign in, click "Add Session". Step I should show a Location field below the host grid.

- [ ] **Step 4: Commit**

```bash
cd /Users/danielfainsinger/Documents/GitHub/experiments
git add freemaltsons/index.html
git commit -m "feat(freemaltsons): add location field to add session wizard step 1"
```

---

### Task 4: Add "Add later" option and "Hide whisky" checkbox to wizard Step 1

**Files:**
- Modify: `index.html:2067` — Step 1 HTML
- Modify: `index.html` — CSS section (before `</style>`)
- Modify: `index.html:2603` — `setupModal()` JS

- [ ] **Step 1: Update Step 1 HTML**

Find the Step 1 block (around line 2067). Replace it entirely with:

```html
    <!-- Step 1: Whisky -->
    <div id="step1" style="display:none">
      <div class="modal-step-label">Step II of III</div>
      <h2 class="modal-heading">What is being poured?</h2>
      <div class="field">
        <label class="field-label">Whisky Name</label>
        <div class="search-wrap">
          <input class="modal-input" id="whiskySearchInput" type="text" placeholder="Search or type a name…" autocomplete="off">
          <div class="autocomplete-list" id="autocompleteList"></div>
        </div>
        <div class="spinner-row" id="fetchSpinner"><div class="spinner"></div> Searching…</div>
      </div>
      <button class="add-later-btn" id="addLaterBtn">Add whisky later →</button>
      <div id="dupWarning" style="display:none; margin-top:12px; padding:10px 14px; background:var(--oxblood); border:1px solid #8b2020; border-radius:var(--radius); color:#f5a0a0; font-family:var(--font-sans); font-size:0.75rem; line-height:1.5;"></div>
      <div id="hideWhiskyRow" style="display:none; margin-top:16px;">
        <label style="display:flex;align-items:center;gap:10px;cursor:pointer;font-family:var(--font-sans);font-size:0.8rem;color:var(--text-dim)">
          <input type="checkbox" id="hideWhiskyCheck" style="width:16px;height:16px;accent-color:var(--gold)">
          Hide whisky from others until the night
        </label>
      </div>
      <div class="modal-actions">
        <button class="btn-back" id="whiskyBackBtn">← Back</button>
        <button class="btn-primary" id="whiskyNextBtn" disabled>Continue</button>
      </div>
    </div>
```

- [ ] **Step 2: Add CSS for the "Add later" button**

Find `</style>` (around line 1942). Before it, add:

```css
/* ── Add later button ──────────────────────────────────────────── */
.add-later-btn {
  display: block;
  width: 100%;
  margin-top: 12px;
  padding: 10px 16px;
  background: none;
  border: 1px dashed var(--border-accent);
  border-radius: var(--radius);
  color: var(--text-dim);
  font-family: var(--font-sans);
  font-size: 0.78rem;
  letter-spacing: 0.05em;
  cursor: pointer;
  text-align: center;
  transition: color var(--transition), border-color var(--transition);
}
.add-later-btn:hover { color: var(--gold); border-color: var(--gold-dim); }
.add-later-btn.selected { color: var(--gold); border-color: var(--gold); background: var(--gold-faint); }
```

- [ ] **Step 3: Wire up "Add later" and "Hide whisky" in `setupModal()`**

Find `setupModal()` (around line 2603). It currently ends like this (around line 2656):
```js
  document.getElementById('imageUrlInput').addEventListener('change', e => {
    const url = e.target.value.trim();
    if (url) { newSession.image_url = url; showImagePreview(url, 'Manually added'); }
  });

  document.getElementById('findImageBtn').addEventListener('click', async () => {
    // ... existing find image logic ...
  });
}  // <-- closing brace of setupModal()
```

Before the closing `}` of `setupModal()`, add:

```js
  document.getElementById('addLaterBtn').addEventListener('click', () => {
    newSession.whisky    = null;
    newSession.region    = null;
    newSession.rrp       = null;
    newSession.image_url = null;
    document.getElementById('whiskySearchInput').value = '';
    document.getElementById('autocompleteList').classList.remove('open');
    document.getElementById('dupWarning').style.display = 'none';
    document.getElementById('addLaterBtn').classList.add('selected');
    document.getElementById('whiskyNextBtn').disabled = false;
  });

  document.getElementById('hideWhiskyCheck').addEventListener('change', e => {
    newSession.whisky_hidden = e.target.checked;
  });
```

- [ ] **Step 4: Reset "Add later" state in `openModal()`**

In `openModal()`, in the existing reset block (around line 2675), add:
```js
document.getElementById('addLaterBtn').classList.remove('selected');
document.getElementById('hideWhiskyCheck').checked = false;
document.getElementById('hideWhiskyRow').style.display = 'none';
newSession.whisky_hidden = false;
```

- [ ] **Step 5: Show "Hide whisky" checkbox in `goStep()` when appropriate**

Find `goStep()` (around line 2693). It currently looks like:
```js
function goStep(n) {
  for (let i = 0; i < 3; i++) {
    document.getElementById('step' + i).style.display = i === n ? 'block' : 'none';
    const dot = document.getElementById('dot' + i);
    dot.classList.toggle('done', i < n);
    dot.classList.toggle('active', i === n);
  }
  if (n === 1) setTimeout(() => document.getElementById('whiskySearchInput').focus(), 80);
}
```

Replace with:
```js
function goStep(n) {
  for (let i = 0; i < 3; i++) {
    document.getElementById('step' + i).style.display = i === n ? 'block' : 'none';
    const dot = document.getElementById('dot' + i);
    dot.classList.toggle('done', i < n);
    dot.classList.toggle('active', i === n);
  }
  if (n === 1) {
    setTimeout(() => document.getElementById('whiskySearchInput').focus(), 80);
    const isOwnSession = newSession.host &&
      currentMember() &&
      newSession.host.toLowerCase() === currentMember().toLowerCase();
    document.getElementById('hideWhiskyRow').style.display = isOwnSession ? '' : 'none';
  }
}
```

- [ ] **Step 6: Clear "Add later" selection when a whisky is chosen**

In `selectWhisky()` (around line 2786), add at the very start of the function body:
```js
  document.getElementById('addLaterBtn').classList.remove('selected');
```

- [ ] **Step 7: Manual verification**

Open `index.html`. Sign in as Flatty. Click "Add Session", pick Flatty as host. In Step II:
- An "Add whisky later →" button appears.
- A "Hide whisky from others until the night" checkbox appears (because Flatty is the signed-in host).
- Clicking "Add later" highlights it and enables Continue.
- Ticking the checkbox updates `newSession.whisky_hidden`.

Sign in as another member. Add a session for yourself:
- The "Hide whisky" checkbox should still appear (you are the host).
Add a session for Flatty:
- The "Hide whisky" checkbox should NOT appear.

- [ ] **Step 8: Commit**

```bash
cd /Users/danielfainsinger/Documents/GitHub/experiments
git add freemaltsons/index.html
git commit -m "feat(freemaltsons): add 'add later' and 'hide whisky' options to wizard step 2"
```

---

## Chunk 3: Card / Grid Display

### Task 5: Upcoming sessions float to top, show "???", show location, show calendar buttons

**Files:**
- Modify: `index.html:2520` — `applyFilters`
- Modify: `index.html:2550` — `renderCard`
- Modify: `index.html` CSS section — new styles

- [ ] **Step 1: Sort upcoming sessions to top in `applyFilters()`**

Find the sort in `applyFilters()` (around line 2522):
```js
  sessions.sort((a, b) => {
    const [ar, as_] = (a.id||':').split(':');
    const [br, bs]  = (b.id||':').split(':');
    const ri = ROMAN.indexOf(br) - ROMAN.indexOf(ar);
    return ri !== 0 ? ri : ROMAN.indexOf(bs) - ROMAN.indexOf(as_);
  });
```
Replace with:
```js
  const todayStr = localToday();
  sessions.sort((a, b) => {
    const aUp = !!(a.date && a.date >= todayStr);
    const bUp = !!(b.date && b.date >= todayStr);
    if (aUp && !bUp) return -1;
    if (!aUp && bUp) return 1;
    // Both upcoming: sort ascending by date
    if (aUp && bUp) return (a.date || '').localeCompare(b.date || '');
    // Both past: existing sort descending by ID
    const [ar, as_] = (a.id||':').split(':');
    const [br, bs]  = (b.id||':').split(':');
    const ri = ROMAN.indexOf(br) - ROMAN.indexOf(ar);
    return ri !== 0 ? ri : ROMAN.indexOf(bs) - ROMAN.indexOf(as_);
  });
```

- [ ] **Step 2: Replace `renderCard()` with upcoming-aware version**

Replace the entire `renderCard(s)` function (lines ~2550–2577) with:

```js
function renderCard(s) {
  const upcoming = isUpcoming(s);
  const isHost   = s.host && currentMember() &&
    s.host.toLowerCase() === currentMember().toLowerCase();
  const hidden   = (s.whisky_hidden || !s.whisky) && !isHost;
  const whiskyDisplay = hidden
    ? `<em style="color:var(--text-faint);letter-spacing:0.15em">???</em>`
    : s.whisky
      ? esc(s.whisky)
      : (isHost
          ? `<em style="color:var(--text-faint)">Not chosen yet</em>`
          : `<em style="color:var(--text-faint)">Unknown</em>`);

  const img = (!hidden && s.image_url)
    ? `<img src="${esc(s.image_url)}" alt="${esc(s.whisky||'')}" onerror="this.closest('.card-image').innerHTML=noImgHtml()">`
    : noImgHtml();

  const badgeText = s.id ? s.id.replace(':', ' · ') : '–';
  const scores    = Object.values(s.ratings || {})
    .map(r => r && r.score != null ? Number(r.score) : null)
    .filter(v => v !== null && v > 0);
  const avgScore  = scores.length
    ? (scores.reduce((a,b) => a+b,0) / scores.length).toFixed(1)
    : null;

  const calendarBtns = upcoming ? `
    <div class="card-calendar-btns" onclick="event.stopPropagation()">
      <button class="cal-btn" onclick="addToGoogleCalendar(${allSessions.indexOf(s)})">Google Cal</button>
      <button class="cal-btn" onclick="downloadIcs(${allSessions.indexOf(s)})">.ics</button>
    </div>` : '';

  return `<div class="card${upcoming ? ' card-upcoming' : ''}" onclick="openDetailModal(${allSessions.indexOf(s)})">
    ${upcoming ? '<div class="card-upcoming-label">Upcoming</div>' : ''}
    <div class="card-image">${img}</div>
    <div class="card-badge-rule">
      <span class="session-badge">${esc(badgeText)}</span>
    </div>
    <div class="card-body">
      <h3 class="card-whisky">${whiskyDisplay}</h3>
      ${!hidden && s.dm_url ? `<a class="card-dm-link" href="${esc(s.dm_url)}" target="_blank" rel="noopener" onclick="event.stopPropagation()">Dan Murphy's ↗</a>` : ''}
      ${!hidden && s.region ? `<p class="card-region">${esc(s.region)}</p>` : '<p class="card-region" style="opacity:0">&nbsp;</p>'}
      <div class="card-divider"></div>
      <div class="card-meta">
        <span class="card-host">${esc(s.host||'')}</span>
        <span style="display:flex;align-items:center;gap:6px">
          ${!upcoming && avgScore ? `<span class="card-score-badge">★ ${avgScore}</span>` : ''}
          ${!hidden && s.rrp ? `<span class="card-rrp">$${s.rrp.toFixed(0)}</span>` : ''}
        </span>
      </div>
      ${s.date ? `<div class="card-date">${formatDate(s.date)}</div>` : ''}
      ${s.location ? `<div class="card-location">📍 ${esc(s.location)}</div>` : ''}
      ${calendarBtns}
    </div>
  </div>`;
}
```

- [ ] **Step 3: Add CSS for upcoming cards and calendar buttons**

Before `</style>`, add:

```css
/* ── Upcoming session cards ────────────────────────────────────── */
.card-upcoming {
  border-color: var(--gold-dim);
  box-shadow: 0 0 0 1px var(--gold-dim), var(--shadow-card);
}
.card-upcoming-label {
  font-family: var(--font-sans);
  font-size: 0.6rem;
  letter-spacing: 0.18em;
  text-transform: uppercase;
  color: var(--gold);
  padding: 6px 12px 0;
}
.card-location {
  font-family: var(--font-sans);
  font-size: 0.72rem;
  color: var(--text-dim);
  margin-top: 4px;
}
.card-calendar-btns {
  display: flex;
  gap: 6px;
  margin-top: 10px;
}
.cal-btn {
  flex: 1;
  padding: 6px 8px;
  background: var(--gold-faint);
  border: 1px solid var(--gold-dim);
  border-radius: var(--radius-sm);
  color: var(--gold);
  font-family: var(--font-sans);
  font-size: 0.68rem;
  letter-spacing: 0.04em;
  cursor: pointer;
  transition: background var(--transition), color var(--transition);
}
.cal-btn:hover { background: var(--border-accent); color: var(--gold-lt); }
```

- [ ] **Step 4: Manual verification**

Create a test session with a future date (you can use the browser console: add a dummy entry to `allSessions` with `date: '2099-01-01'` and call `applyFilters()`). Confirm:
- It appears at the top of the grid with a gold border and "Upcoming" label.
- Non-host sees "???".
- Calendar buttons appear and don't propagate the click to the card.
- Location displays beneath the date if set.

- [ ] **Step 5: Commit**

```bash
cd /Users/danielfainsinger/Documents/GitHub/experiments
git add freemaltsons/index.html
git commit -m "feat(freemaltsons): upcoming cards float to top, show ???, location and calendar buttons"
```

---

## Chunk 4: Calendar Functions

### Task 6: Implement Google Calendar and .ics export

**Files:**
- Modify: `index.html` — add after `formatDate()` (~line 2600)

- [ ] **Step 1: Add calendar helper functions**

After the `formatDate()` function (around line 2600), add:

```js
// ─── Calendar Export ──────────────────────────────────────────────────
function calendarEventTitle(s) {
  return `Freemaltsons ${s.id || ''} @ ${s.host || ''}`.trim();
}

function calendarDescription(s) {
  const isHost    = s.host && currentMember() &&
    s.host.toLowerCase() === currentMember().toLowerCase();
  const showWhisky = s.whisky && (!s.whisky_hidden || isHost);
  return showWhisky ? s.whisky : 'Whisky TBA';
}

function addToGoogleCalendar(idx) {
  const s = allSessions[idx];
  if (!s) return;
  const date    = (s.date || '').replace(/-/g, '');
  const nextDay = s.date
    ? new Date(new Date(s.date + 'T00:00:00').getTime() + 86400000)
        .toISOString().slice(0,10).replace(/-/g,'')
    : date;
  const url = 'https://calendar.google.com/calendar/render?action=TEMPLATE'
    + '&text='     + encodeURIComponent(calendarEventTitle(s))
    + '&dates='    + date + '/' + nextDay
    + '&details='  + encodeURIComponent(calendarDescription(s))
    + '&location=' + encodeURIComponent(s.location || '');
  window.open(url, '_blank', 'noopener');
}

function downloadIcs(idx) {
  const s = allSessions[idx];
  if (!s) return;
  const date    = (s.date || '').replace(/-/g, '');
  const nextDay = s.date
    ? new Date(new Date(s.date + 'T00:00:00').getTime() + 86400000)
        .toISOString().slice(0,10).replace(/-/g,'')
    : date;
  const uid  = 'freemaltsons-' + (s.id || Date.now()) + '@freemaltsons';
  const loc  = s.location || '';
  const ics  = [
    'BEGIN:VCALENDAR',
    'VERSION:2.0',
    'PRODID:-//Freemaltsons//Whisky Nights//EN',
    'BEGIN:VEVENT',
    'UID:'    + uid,
    'DTSTART;VALUE=DATE:' + date,
    'DTEND;VALUE=DATE:'   + nextDay,
    'SUMMARY:'     + calendarEventTitle(s),
    'DESCRIPTION:' + calendarDescription(s),
    loc ? 'LOCATION:' + loc : '',
    'END:VEVENT',
    'END:VCALENDAR',
  ].filter(Boolean).join('\r\n');

  const blob = new Blob([ics], { type: 'text/calendar' });
  const a    = Object.assign(document.createElement('a'), {
    href:     URL.createObjectURL(blob),
    download: 'freemaltsons-' + (s.id || 'session').replace(':', '-') + '.ics',
  });
  a.click();
  URL.revokeObjectURL(a.href);
}
```

- [ ] **Step 2: Manual verification**

Create an upcoming session and click "Google Cal" — a Google Calendar tab should open with event title `Freemaltsons X:I @ Flatty`, the correct date, and location. Click ".ics" — a `.ics` file downloads and opens in Calendar/Outlook with the same details.

- [ ] **Step 3: Commit**

```bash
cd /Users/danielfainsinger/Documents/GitHub/experiments
git add freemaltsons/index.html
git commit -m "feat(freemaltsons): add Google Calendar and .ics export"
```

---

## Chunk 5: Detail Modal

### Task 7: Update detail modal for upcoming sessions

**Files:**
- Modify: `index.html:2144` — Detail modal HTML
- Modify: `index.html:3313` — `openDetailModal` JS

- [ ] **Step 1: Add new HTML elements to detail modal**

Find the detail modal HTML (around line 2154, inside `<div id="detailDetailsPane">`).

**A.** After `<div class="detail-meta" id="detailMeta"></div>`, add:

```html
        <div id="detailCalendarBtns" style="margin:12px 0;gap:8px;display:none">
          <button class="cal-btn" id="detailGoogleCalBtn">Add to Google Calendar</button>
          <button class="cal-btn" id="detailIcsBtn">Download .ics</button>
        </div>
        <div id="detailHostActions" style="display:none; margin-bottom:16px;">
          <button class="btn-primary" id="detailRevealBtn" style="display:none">Reveal Whisky</button>
          <div id="detailInlineWhiskySearch" style="display:none">
            <div class="field">
              <label class="field-label">Add Whisky</label>
              <div class="search-wrap">
                <input class="modal-input" id="detailWhiskySearchInput" type="text" placeholder="Search or type a name…" autocomplete="off">
                <div class="autocomplete-list" id="detailAutocompleteList"></div>
              </div>
            </div>
          </div>
        </div>
```

**B.** Find the `<div class="detail-actions">` at the bottom of `detailDetailsPane` (around line 2174):
```html
        <div class="detail-actions">
          <button class="btn-back" id="detailEditBtn">Edit Session</button>
        </div>
```
Add an `id` to the wrapper:
```html
        <div class="detail-actions" id="detailActionsRow">
          <button class="btn-back" id="detailEditBtn">Edit Session</button>
        </div>
```

- [ ] **Step 2: Replace `openDetailModal()` with upcoming-aware version**

Replace the entire `openDetailModal()` function (lines ~3313–3384) with:

```js
function openDetailModal(idx) {
  const s = allSessions[idx];
  if (!s) return;
  detailSessionIdx = idx;

  const upcoming = isUpcoming(s);
  const isHost   = s.host && currentMember() &&
    s.host.toLowerCase() === currentMember().toLowerCase();
  const hidden   = (s.whisky_hidden || !s.whisky) && !isHost;

  // Always open on Details pane
  switchDetailPane(false);
  if (!upcoming) renderRatingsPaneLocked(s);

  // Hide Ratings tab for upcoming sessions
  document.getElementById('detailTabRatings').style.display = upcoming ? 'none' : '';

  // Hero image
  const hero = document.getElementById('detailHero');
  if (!hidden && s.image_url) {
    hero.innerHTML = `<img src="${esc(s.image_url)}" alt="${esc(s.whisky||'')}" onerror="this.parentElement.innerHTML='<div class=\\'detail-hero-empty\\'>🥃</div>'">`;
  } else {
    hero.innerHTML = `<div class="detail-hero-empty">🥃</div>`;
  }

  // Badge & whisky name
  document.getElementById('detailBadge').textContent  = s.id ? s.id.replace(':', ' · ') : '';
  document.getElementById('detailWhisky').textContent = hidden
    ? '???'
    : (s.whisky || (isHost ? 'Not chosen yet' : '–'));

  // Meta
  document.getElementById('detailMeta').innerHTML = [
    s.host     ? `<div class="detail-meta-item"><strong>Host</strong>${esc(s.host)}</div>`                  : '',
    s.date     ? `<div class="detail-meta-item"><strong>Date</strong>${formatDate(s.date)}</div>`            : '',
    s.location ? `<div class="detail-meta-item"><strong>Location</strong>${esc(s.location)}</div>`          : '',
    (!hidden && s.region) ? `<div class="detail-meta-item"><strong>Region</strong>${esc(s.region)}</div>`   : '',
    (!hidden && s.rrp)    ? `<div class="detail-meta-item"><strong>RRP</strong>$${esc(String(s.rrp))}</div>` : '',
  ].join('');

  // Calendar buttons (upcoming only)
  const calBtns = document.getElementById('detailCalendarBtns');
  if (upcoming) {
    calBtns.style.display = 'flex';
    document.getElementById('detailGoogleCalBtn').onclick = () => addToGoogleCalendar(idx);
    document.getElementById('detailIcsBtn').onclick       = () => downloadIcs(idx);
  } else {
    calBtns.style.display = 'none';
  }

  // Host-only actions: Reveal / Add whisky
  // Note: whisky:null + whisky_hidden:true is valid — show "Add whisky" only in that case
  const hostActions  = document.getElementById('detailHostActions');
  const revealBtn    = document.getElementById('detailRevealBtn');
  const inlineSearch = document.getElementById('detailInlineWhiskySearch');
  if (isHost && upcoming) {
    hostActions.style.display = '';
    if (s.whisky && s.whisky_hidden) {
      // Has whisky but hidden — show Reveal button
      revealBtn.style.display    = '';
      inlineSearch.style.display = 'none';
    } else if (!s.whisky) {
      // No whisky yet (with or without whisky_hidden) — show inline search
      revealBtn.style.display    = 'none';
      inlineSearch.style.display = '';
      setupDetailWhiskySearch(idx);
    } else {
      // Whisky set, not hidden — nothing to do
      hostActions.style.display = 'none';
    }
  } else {
    hostActions.style.display = 'none';
  }

  revealBtn.onclick = async () => {
    allSessions[idx].whisky_hidden = false;
    revealBtn.disabled    = true;
    revealBtn.textContent = 'Revealing…';
    await githubSave();
    closeDetailModal();
    applyFilters();
    renderHeaderStats();
    renderLedger();
  };

  // Distillery section (skip if whisky is hidden)
  const distName    = (!hidden && s.whisky) ? extractDistilleryName(s.whisky) : null;
  const distInfo    = distName ? DISTILLERY_COORDS[distName] : null;
  const distSection = document.getElementById('detailDistillerySection');
  if (distName) {
    distSection.style.display = '';
    document.getElementById('detailDistilleryName').textContent = distName + ' Distillery';
    document.getElementById('detailWikiExtract').textContent    = 'Loading…';
    document.getElementById('detailDistilleryLinks').innerHTML  = distInfo?.url
      ? `<a class="detail-link" href="${esc(distInfo.url)}" target="_blank" rel="noopener">Visit Distillery →</a>`
      : '';
    fetchWiki(distName).then(wiki => {
      const extractEl = document.getElementById('detailWikiExtract');
      if (!wiki?.extract) { extractEl.textContent = ''; return; }
      let text = wiki.extract;
      if (text.length > 300) {
        const cut = text.lastIndexOf('.', 300);
        text = cut > 80 ? text.slice(0, cut + 1) : text.slice(0, 300) + '…';
      }
      extractEl.textContent = text;
      if (wiki.wikiUrl) {
        const a = Object.assign(document.createElement('a'), {
          className: 'detail-link', href: wiki.wikiUrl,
          target: '_blank', rel: 'noopener', textContent: 'Wikipedia →'
        });
        document.getElementById('detailDistilleryLinks').appendChild(a);
      }
    });
  } else {
    distSection.style.display = 'none';
  }

  // Find links (skip if hidden)
  if (!hidden && s.whisky) {
    const searchUrl = `https://www.google.com/search?q=${encodeURIComponent(s.whisky + ' whisky')}`;
    document.getElementById('detailWhiskyLinks').innerHTML = [
      s.dm_url ? `<a class="detail-link" href="${esc(s.dm_url)}" target="_blank" rel="noopener">Dan Murphy's →</a>` : '',
      `<a class="detail-link" href="${esc(searchUrl)}" target="_blank" rel="noopener">Search Online →</a>`,
    ].join('');
  } else {
    document.getElementById('detailWhiskyLinks').innerHTML = '';
  }

  // Edit button — host only
  document.getElementById('detailEditBtn').style.display = isHost ? '' : 'none';

  document.getElementById('detailModalOverlay').classList.add('open');
}
```

- [ ] **Step 3: Add `setupDetailWhiskySearch()` and `selectDetailWhisky()` functions**

Immediately after `openDetailModal()`, add:

```js
function setupDetailWhiskySearch(sessionIdx) {
  const input = document.getElementById('detailWhiskySearchInput');
  const list  = document.getElementById('detailAutocompleteList');
  input.value  = '';
  list.innerHTML = '';
  list.classList.remove('open');

  let t = null;
  input.oninput = () => {
    const q = input.value.trim();
    clearTimeout(t);
    if (q.length < 2) { list.classList.remove('open'); return; }
    t = setTimeout(() => {
      const results = searchWhiskyLocal(q);
      const items   = results.map(r =>
        `<div class="autocomplete-item" onclick='selectDetailWhisky(${sessionIdx}, ${JSON.stringify(JSON.stringify(r))})'>
          ${r.image_url ? `<img src="${esc(r.image_url)}" onerror="this.style.display='none'">` : ''}
          <div class="ac-info">
            <div class="ac-name">${esc(r.whisky)}</div>
            <div class="ac-meta">${[r.region, r.rrp ? '$'+r.rrp : null].filter(Boolean).join(' · ')}</div>
          </div>
        </div>`).join('');
      const custom =
        `<div class="autocomplete-item" onclick='selectDetailWhisky(${sessionIdx}, ${JSON.stringify(JSON.stringify({ whisky: q, region: null, rrp: null, image_url: null }))})'>
          <div class="ac-info"><div class="ac-name">${esc(q)}</div><div class="ac-meta">Enter as new</div></div>
        </div>`;
      list.innerHTML = items + custom;
      list.classList.add('open');
    }, 150);
  };
}

async function selectDetailWhisky(sessionIdx, itemJson) {
  const item = JSON.parse(itemJson);
  const s    = allSessions[sessionIdx];
  if (!s) return;
  s.whisky    = item.whisky;
  s.region    = item.region    || s.region;
  s.rrp       = item.rrp       || s.rrp;
  s.image_url = item.image_url || s.image_url;
  await githubSave();
  applyFilters();
  openDetailModal(sessionIdx);
}
```

- [ ] **Step 4: Manual verification**

Create an upcoming session as Flatty with hidden whisky and location:
- Other members see "???" and the location. Ratings tab is hidden. Edit button is hidden.
- Flatty sees the real whisky name (or "Not chosen yet"), Reveal button, and Edit button.
- Clicking Reveal saves and re-renders: all members now see the whisky.
- For a session with no whisky, Flatty sees inline search; selecting a whisky saves it.
- Calendar buttons in the detail modal work correctly.

- [ ] **Step 5: Commit**

```bash
cd /Users/danielfainsinger/Documents/GitHub/experiments
git add freemaltsons/index.html
git commit -m "feat(freemaltsons): update detail modal — location, reveal, inline whisky search, host gating"
```

---

## Chunk 6: Edit Modal

### Task 8: Add location, hide whisky toggle, and allow null whisky in edit modal

**Files:**
- Modify: `index.html:2196` — Edit modal HTML
- Modify: `index.html:2914` — `openEditModal`
- Modify: `index.html:2949` — `saveEditSession`

- [ ] **Step 1: Add Location field to edit modal HTML**

Find the Edit Modal (around line 2196). After the Whisky `<div class="field">` block, add:
```html
    <div class="field">
      <label class="field-label">Location <span class="field-note">optional</span></label>
      <input class="modal-input" id="editLocationInput" type="text" placeholder="e.g. Flatty's place">
    </div>
```

- [ ] **Step 2: Add "Hide whisky" toggle to edit modal HTML**

After the Dan Murphy's URL field block (around line 2242), add:
```html
    <div style="margin: 16px 0;">
      <label style="display:flex;align-items:center;gap:10px;cursor:pointer;font-family:var(--font-sans);font-size:0.8rem;color:var(--text-dim)">
        <input type="checkbox" id="editHideWhiskyCheck" style="width:16px;height:16px;accent-color:var(--gold)">
        Hide whisky from others until the night
      </label>
    </div>
```

- [ ] **Step 3: Pre-fill new fields in `openEditModal()`**

In `openEditModal()` (around line 2914), after the existing field assignments, add:
```js
  document.getElementById('editLocationInput').value     = s.location || '';
  document.getElementById('editHideWhiskyCheck').checked = s.whisky_hidden || false;
```

- [ ] **Step 4: Save new fields in `saveEditSession()`**

Find the `fields` object in `saveEditSession()` (around line 2955). Replace it with:
```js
  const fields = {
    whisky:        document.getElementById('editWhiskyInput').value.trim() || null,
    region:        document.getElementById('editRegionInput').value.trim() || null,
    rrp:           parseFloat(document.getElementById('editRrpInput').value) || null,
    date:          document.getElementById('editDateInput').value || null,
    image_url:     document.getElementById('editImageUrlInput').value.trim() || null,
    dm_url:        document.getElementById('editDmUrlInput').value.trim() || null,
    location:      document.getElementById('editLocationInput').value.trim() || null,
    whisky_hidden: document.getElementById('editHideWhiskyCheck').checked,
  };
```

- [ ] **Step 5: Manual verification**

Open an upcoming session as the host. Click Edit:
- Location field is pre-filled and saves correctly.
- "Hide whisky" checkbox reflects and updates `whisky_hidden`.
- Clearing the whisky field and saving sets `whisky: null` (check the saved JSON).

- [ ] **Step 6: Commit**

```bash
cd /Users/danielfainsinger/Documents/GitHub/experiments
git add freemaltsons/index.html
git commit -m "feat(freemaltsons): add location and whisky_hidden to edit modal"
```

---

## Chunk 7: Final Integration

### Task 9: End-to-end test and push

- [ ] **Step 1: Full end-to-end test**

Run through the complete flow:
1. Sign in as Flatty. Add a session: host=Flatty, date=future, location="Flatty's place", whisky hidden. Confirm saves to GitHub.
2. Card appears at top of grid with gold border, "Upcoming" label, "???", location, calendar buttons.
3. Click card — detail modal shows location, Reveal button (host), calendar buttons. Ratings tab hidden.
4. Sign out and sign in as another member. Confirm "???", no Edit button, no Reveal button.
5. Sign back in as Flatty. Click Reveal — whisky revealed to all members.
6. Flatty adds a session with "Add later" (null whisky). Card shows "Not chosen yet" to Flatty, "???" to others. Inline whisky search works in detail modal.
7. Google Calendar and .ics export produce correct event titles.
8. Header stats and ledger exclude upcoming sessions.
9. Edit modal shows location and hide whisky fields for host.

- [ ] **Step 2: Push to GitHub**

```bash
cd /Users/danielfainsinger/Documents/GitHub/experiments
git stash && git pull --rebase origin main && git stash pop
git push origin main
```
