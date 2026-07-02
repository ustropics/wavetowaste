# Lazy-load MC dataset in wave_to_waste.html Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Render `wave_to_waste.html`'s map/UI immediately on load, and load the 227MB MC dataset in the background afterward, so the page is usable before the download finishes.

**Architecture:** Split the current blocking bootstrap into two phases — an immediate `init()` call after the small nesting-zones file loads (empty `allParticles`), and a background `startMcLoadInBackground()` that loads the MC dataset via a single shared promise (deduped so a same-model button click during the background load can't trigger a second, duplicate download), shows a small corner badge while in flight, and applies the data (filter pills, layers, map bounds) once ready.

**Tech Stack:** Vanilla JS, Leaflet, single static HTML file. No build step, no test framework — verification is done by driving the page with a headless Chromium (`playwright-core`) script and asserting on `window` state / DOM, the same approach used earlier in this session to debug this file.

**Deviation from spec's `pendingFilterState`:** the design doc proposed a `pendingFilterState` object to capture filter-pill/model clicks made before data arrives. In practice `rebuildFilterPills()` (existing code) builds year/month pills from `allParticles`, which is empty until MC data lands — so there are no real year/month pills to click yet (nothing to queue). The only meaningful "early interaction" is clicking a model button before MC data is ready, which Task 5/6's shared `mcLoadPromise` already handles (the click `await`s the same in-flight load instead of queuing a separate state object). A separate `pendingFilterState` would have no reachable code path to exercise, so it's dropped as unnecessary (YAGNI).

---

### Task 1: Add badge CSS

**Files:**
- Modify: `wave_to_waste.html:68` (right after the existing `.loader-msg` rule, before the `/* ── Navbar ─────` comment)

- [ ] **Step 1: Insert badge styles**

Current text at line 68:
```css
  .loader-msg { font-size: 13px; color: var(--text-secondary); font-family: 'JetBrains Mono', monospace; }

  /* ── Navbar ─────────────────────────────────────── */
```

Replace with:
```css
  .loader-msg { font-size: 13px; color: var(--text-secondary); font-family: 'JetBrains Mono', monospace; }

  /* ── Background MC data-load badge ──────────────── */
  #mcLoadBadge {
    display: none; position: fixed; bottom: 16px; right: 16px; z-index: 1200;
    background: var(--bg-panel); border: 1px solid rgba(255,255,255,0.13);
    border-radius: 8px; padding: 8px 14px; box-shadow: 0 4px 20px rgba(0,0,0,0.45);
    font-family: 'JetBrains Mono', monospace; font-size: 12px; color: var(--text-secondary);
    align-items: center; gap: 8px;
  }
  #mcLoadBadge.visible { display: flex; }
  #mcLoadBadge.error { color: var(--danger); border-color: rgba(255,107,107,0.4); }
  #mcLoadBadge .badge-spinner {
    width: 12px; height: 12px; border-radius: 50%;
    border: 2px solid rgba(0, 136, 122, 0.2); border-top-color: #00887a;
    animation: spin 0.8s linear infinite; flex-shrink: 0;
  }
  #mcLoadBadge.error .badge-spinner { display: none; }

  /* ── Navbar ─────────────────────────────────────── */
```

`var(--bg-panel)`, `var(--danger)`, `var(--text-secondary)`, and the `spin` keyframe all already exist earlier in the same `<style>` block (used by `#mobilePanelFab` and `.loader-spinner`) — no new custom properties needed.

- [ ] **Step 2: No test yet — markup doesn't exist. Continue to Task 2.**

---

### Task 2: Add badge markup

**Files:**
- Modify: `wave_to_waste.html:527-531`

- [ ] **Step 1: Insert badge div after the loader**

Current text:
```html
<!-- ── Loader ───────────────────────────────────────────────────────────── -->
<div id="loader">
  <div class="loader-spinner"></div>
  <div class="loader-msg" id="loaderMsg">Loading data&hellip;</div>
</div>
```

Replace with:
```html
<!-- ── Loader ───────────────────────────────────────────────────────────── -->
<div id="loader">
  <div class="loader-spinner"></div>
  <div class="loader-msg" id="loaderMsg">Loading data&hellip;</div>
</div>

<!-- ── Background MC data-load badge ───────────────────────────────────── -->
<div id="mcLoadBadge">
  <div class="badge-spinner"></div>
  <span id="mcLoadBadgeMsg">Loading MC data&hellip;</span>
</div>
```

- [ ] **Step 2: Commit**

```bash
git add wave_to_waste.html
git commit -m "Add background data-load badge markup and styles"
```

---

### Task 3: Add state vars and badge helper functions

**Files:**
- Modify: `wave_to_waste.html:928-932` (state vars)
- Modify: `wave_to_waste.html:1125-1133` (right after `showLoader`)

- [ ] **Step 1: Add background-load state vars**

Current text at line 928-932:
```js
let currentModel = 'mc';
let allParticles  = [];
```
(with `_filtersInitialized` declared a couple lines below)

Find:
```js
let currentModel = 'mc';
let allParticles  = [];
```

Replace with:
```js
let currentModel = 'mc';
let allParticles  = [];
let mcLoadPromise = null;
let mcDataReady   = false;
```

- [ ] **Step 2: Add badge helper functions**

Current text at line 1125-1133:
```js
function showLoader(visible, msg) {
  const el = document.getElementById('loader');
  if (visible) {
    el.classList.remove('hidden');
    if (msg) document.getElementById('loaderMsg').textContent = msg;
  } else {
    el.classList.add('hidden');
  }
}
```

Replace with:
```js
function showLoader(visible, msg) {
  const el = document.getElementById('loader');
  if (visible) {
    el.classList.remove('hidden');
    if (msg) document.getElementById('loaderMsg').textContent = msg;
  } else {
    el.classList.add('hidden');
  }
}

function showMcBadge(msg, isError) {
  const el = document.getElementById('mcLoadBadge');
  document.getElementById('mcLoadBadgeMsg').textContent = msg;
  el.classList.add('visible');
  el.classList.toggle('error', !!isError);
}
function hideMcBadge() {
  document.getElementById('mcLoadBadge').classList.remove('visible');
}
```

- [ ] **Step 3: Commit**

```bash
git add wave_to_waste.html
git commit -m "Add background MC load state vars and badge helpers"
```

---

### Task 4: Extract fitBounds into a reusable helper

**Files:**
- Modify: `wave_to_waste.html:1451` (add helper right after `rebuildLayers`)
- Modify: `wave_to_waste.html:2284-2300` (`init()` — use the new helper)

- [ ] **Step 1: Add `_fitBoundsToParticles` helper**

Current text at line 1451:
```js
function rebuildLayers() { rebuildHeatmaps(); rebuildDots(); rebuildTrajectories(); rebuildStartPoints(); updateStatsBar(getFiltered()); }
```

Replace with:
```js
function rebuildLayers() { rebuildHeatmaps(); rebuildDots(); rebuildTrajectories(); rebuildStartPoints(); updateStatsBar(getFiltered()); }

function _fitBoundsToParticles(pts) {
  if (!pts.length) return;
  let minLat = Infinity, maxLat = -Infinity, minLon = Infinity, maxLon = -Infinity;
  for (const p of pts) {
    if (p.lat < minLat) minLat = p.lat;
    if (p.lat > maxLat) maxLat = p.lat;
    if (p.lon < minLon) minLon = p.lon;
    if (p.lon > maxLon) maxLon = p.lon;
  }
  map.fitBounds([[minLat-0.3, minLon-0.3],[maxLat+0.3, maxLon+0.3]]);
}
```

- [ ] **Step 2: Simplify `init()` to use the helper**

Current text at line 2284-2300:
```js
function init() {
  _wireSliders();
  rebuildFilterPills();
  rebuildLayers();
  renderNestingZones();
  renderGmuZones();
  if (allParticles.length > 0) {
    let minLat = Infinity, maxLat = -Infinity, minLon = Infinity, maxLon = -Infinity;
    for (const p of allParticles) {
      if (p.lat < minLat) minLat = p.lat;
      if (p.lat > maxLat) maxLat = p.lat;
      if (p.lon < minLon) minLon = p.lon;
      if (p.lon > maxLon) maxLon = p.lon;
    }
    map.fitBounds([[minLat-0.3, minLon-0.3],[maxLat+0.3, maxLon+0.3]]);
  }
}
```

Replace with:
```js
function init() {
  _wireSliders();
  rebuildFilterPills();
  rebuildLayers();
  renderNestingZones();
  renderGmuZones();
  _fitBoundsToParticles(allParticles);
}
```

- [ ] **Step 3: Commit**

```bash
git add wave_to_waste.html
git commit -m "Extract particle bounds-fitting into reusable helper"
```

---

### Task 5: Add `startMcLoadInBackground()`

**Files:**
- Modify: `wave_to_waste.html` (add the function right after `_fitBoundsToParticles`, from Task 4)

- [ ] **Step 1: Add the function**

Insert immediately after the `_fitBoundsToParticles` function added in Task 4:

```js
function _applyMcData() {
  allParticles = (window._FLORIDA_DATA.mc || {}).particles || [];
  allParticles.forEach(p => { p.nesting_beach = nearestNestingBeach(p.lat, p.lon); });
  _buildModelIndex('mc');
  rebuildFilterPills();
  rebuildLayers();
  _fitBoundsToParticles(allParticles);
}

function startMcLoadInBackground() {
  if (mcLoadPromise) return mcLoadPromise;
  showMcBadge('Loading MC data…', false);
  mcLoadPromise = _loadDataset('mc').then(() => {
    mcDataReady = true;
    hideMcBadge();
    if (currentModel === 'mc') _applyMcData();
  }).catch(err => {
    console.error(err);
    showMcBadge('Failed to load data. Make sure florida_mc_data_*.js and florida_nesting_zones.js are in the same folder as this HTML file.', true);
    mcLoadPromise = null; // allow retry
  });
  return mcLoadPromise;
}
```

`_applyMcData()` is pulled out as its own function because both the background-load success path and `switchModel('mc')` (Task 6) need to run the same "data is ready, populate UI" logic.

- [ ] **Step 2: Commit**

```bash
git add wave_to_waste.html
git commit -m "Add background MC dataset loader with shared in-flight promise"
```

---

### Task 6: Route `switchModel('mc')` through the shared promise

**Files:**
- Modify: `wave_to_waste.html:1138-1181`

- [ ] **Step 1: Rewrite `switchModel`**

Current text at line 1138-1181:
```js
async function switchModel(key) {
  if (key === currentModel && allParticles.length > 0) return;
  showLoader(true, 'Loading ' + key.toUpperCase() + ' data…');
  try {
    if (!window._FLORIDA_DATA[key] || !window._FLORIDA_DATA[key]._fully_loaded) {
      await _loadDataset(key);
    }
    currentModel = key;
    allParticles = (window._FLORIDA_DATA[key] || {}).particles || [];

    // Assign nesting beach names for freshly loaded data
    if (nestingPolygons.length > 0) {
      allParticles.forEach(p => { if (!('nesting_beach' in p)) p.nesting_beach = nearestNestingBeach(p.lat, p.lon); });
    }
    _buildModelIndex(key);

    // Update navbar model buttons
    document.getElementById('btnModelMc').classList.toggle('active', key === 'mc');
    document.getElementById('btnModelDet').classList.toggle('active', key === 'det');
    document.getElementById('btnModelStoch').classList.toggle('active', key === 'stoch');

    // Show/hide trajectory toggle (MC only)
    document.getElementById('trajToggleRow').style.display = key === 'mc' ? '' : 'none';
    if (key !== 'mc') {
      document.getElementById('togTrajectories').checked = false;
      _clearSelectedTraj();
      if (trajLayer) { map.removeLayer(trajLayer); trajLayer = null; }
      trajSegments.clear(); selectedPid = null;
    }

    // Update panel title
    document.getElementById('panelTitle').textContent = MODEL_LABELS[key];

    hideChart();
    rebuildFilterPills();
    rebuildLayers();
    _syncDrawerState();
  } catch (err) {
    console.error(err);
    showLoader(true, 'Failed to load data. Check that florida_' + key + '_data.js is in the same folder as this HTML.');
    return;
  }
  showLoader(false);
}
```

Replace with:
```js
async function switchModel(key) {
  if (key === currentModel && allParticles.length > 0) return;

  if (key === 'mc') {
    if (!mcDataReady) {
      await startMcLoadInBackground();
      if (!mcDataReady) return; // load failed; badge already shows the error
    }
    currentModel = 'mc';
    allParticles = (window._FLORIDA_DATA.mc || {}).particles || [];
  } else {
    showLoader(true, 'Loading ' + key.toUpperCase() + ' data…');
    try {
      if (!window._FLORIDA_DATA[key] || !window._FLORIDA_DATA[key]._fully_loaded) {
        await _loadDataset(key);
      }
    } catch (err) {
      console.error(err);
      showLoader(true, 'Failed to load data. Check that florida_' + key + '_data.js is in the same folder as this HTML.');
      return;
    }
    currentModel = key;
    allParticles = (window._FLORIDA_DATA[key] || {}).particles || [];
  }

  // Assign nesting beach names for freshly loaded data
  if (nestingPolygons.length > 0) {
    allParticles.forEach(p => { if (!('nesting_beach' in p)) p.nesting_beach = nearestNestingBeach(p.lat, p.lon); });
  }
  _buildModelIndex(key);

  // Update navbar model buttons
  document.getElementById('btnModelMc').classList.toggle('active', key === 'mc');
  document.getElementById('btnModelDet').classList.toggle('active', key === 'det');
  document.getElementById('btnModelStoch').classList.toggle('active', key === 'stoch');

  // Show/hide trajectory toggle (MC only)
  document.getElementById('trajToggleRow').style.display = key === 'mc' ? '' : 'none';
  if (key !== 'mc') {
    document.getElementById('togTrajectories').checked = false;
    _clearSelectedTraj();
    if (trajLayer) { map.removeLayer(trajLayer); trajLayer = null; }
    trajSegments.clear(); selectedPid = null;
  }

  // Update panel title
  document.getElementById('panelTitle').textContent = MODEL_LABELS[key];

  hideChart();
  rebuildFilterPills();
  rebuildLayers();
  _syncDrawerState();
  showLoader(false);
}
```

Why `switchModel('mc')` still calls `rebuildFilterPills()`/`rebuildLayers()` itself even though `_applyMcData()` (Task 5) might already have: `mcLoadPromise` is cached, so on every subsequent switch back to MC after the first load, `_loadDataset` isn't re-run and `_applyMcData`'s `.then()` doesn't fire again — `switchModel` is the only thing that will rebuild the UI for that click. The one narrow case where both run (user clicks the already-active MC button while the very first background load is still in flight) just means `rebuildFilterPills`/`rebuildLayers` run twice back-to-back — wasted CPU cycles, not a correctness issue (no data is duplicated; `_loadDataset` dedupes via the shared promise, not the UI rebuild).

- [ ] **Step 2: Commit**

```bash
git add wave_to_waste.html
git commit -m "Route switchModel('mc') through the shared background-load promise"
```

---

### Task 7: Rewire bootstrap to not block on the MC dataset

**Files:**
- Modify: `wave_to_waste.html:2302-2320`

- [ ] **Step 1: Rewrite the bootstrap IIFE**

Current text at line 2302-2320:
```js
// ── Bootstrap ──────────────────────────────────────────────────────────────
(async function() {
  try {
    await _loadScript('florida_nesting_zones.js');
    await _loadDataset('mc');
    buildNestingIndex();
    allParticles = (window._FLORIDA_DATA.mc || {}).particles || [];
    allParticles.forEach(p => { p.nesting_beach = nearestNestingBeach(p.lat, p.lon); });
    _buildModelIndex('mc');
    init();
    _initDrawerBasemapList();
    _syncDrawerState();
    showLoader(false);
  } catch(err) {
    console.error(err);
    document.getElementById('loaderMsg').textContent =
      'Failed to load data. Make sure florida_mc_data_*.js and florida_nesting_zones.js are in the same folder as this HTML file.';
  }
})();
```

Replace with:
```js
// ── Bootstrap ──────────────────────────────────────────────────────────────
(async function() {
  try {
    await _loadScript('florida_nesting_zones.js');
    buildNestingIndex();
    init();
    _initDrawerBasemapList();
    _syncDrawerState();
    showLoader(false);
    startMcLoadInBackground();
  } catch(err) {
    console.error(err);
    document.getElementById('loaderMsg').textContent =
      'Failed to load data. Make sure florida_nesting_zones.js is in the same folder as this HTML file.';
  }
})();
```

Note the error message here now only mentions `florida_nesting_zones.js`, since that's the only thing this particular `try/catch` still awaits — a failure to load the MC dataset itself now surfaces through the badge (`showMcBadge`) added in Task 5, not this block.

- [ ] **Step 2: Commit**

```bash
git add wave_to_waste.html
git commit -m "Load MC dataset in background instead of blocking initial render"
```

---

### Task 8: Manual verification (headless browser)

**Files:**
- Create (scratch, not committed): `/private/tmp/claude-501/-Users-ustropics-Documents-tess-wavetowaste/02664d3c-9afe-424a-9d82-d1b4fa3ebaad/scratchpad/verify_lazy_load.mjs`

There is no test framework in this repo (static HTML, no `package.json`). Verification is done by driving the page with a headless Chromium instance via `playwright-core`, matching the approach already used earlier in this session (`scratchpad/test3.mjs`).

- [ ] **Step 1: Confirm the scratchpad already has `playwright-core` installed**

Run: `cd /private/tmp/claude-501/-Users-ustropics-Documents-tess-wavetowaste/02664d3c-9afe-424a-9d82-d1b4fa3ebaad/scratchpad && node -e "require('playwright-core'); console.log('ok')"`
Expected: `ok`

If it prints `Cannot find module`, run `npm install playwright-core@1.61.1 --no-save` in that directory first.

- [ ] **Step 2: Write the verification script**

```js
import { chromium } from 'playwright-core';
import path from 'path';

const execPath = '/Users/ustropics/Library/Caches/ms-playwright/chromium-1223/chrome-mac-x64/Google Chrome for Testing.app/Contents/MacOS/Google Chrome for Testing';
const browser = await chromium.launch({ executablePath: execPath, args: ['--no-sandbox'] });
const page = await browser.newPage();
const msgs = [];
page.on('console', m => msgs.push(`[${m.type()}] ${m.text()}`));
page.on('pageerror', e => msgs.push(`[pageerror] ${e.message}`));

const filePath = 'file://' + path.resolve('/Users/ustropics/Documents/tess/wavetowaste/wave_to_waste.html');
await page.goto(filePath, { waitUntil: 'load', timeout: 60000 });

// ── Check 1: full-screen loader hides almost immediately (doesn't wait on MC data) ──
await page.waitForFunction(() => document.getElementById('loader').classList.contains('hidden'), { timeout: 10000 });
const mcCountAtInit = await page.evaluate(() => (window._FLORIDA_DATA.mc || {}).particles?.length ?? 0);
console.log('CHECK1 loader hidden early, mc particles at that point:', mcCountAtInit);

// ── Check 2: badge is visible while MC data is loading ──
const badgeVisibleEarly = await page.locator('#mcLoadBadge').evaluate(el => el.classList.contains('visible'));
console.log('CHECK2 badge visible while loading:', badgeVisibleEarly);

// ── Check 3: clicking the already-active MC button during background load doesn't throw or duplicate data ──
await page.click('#btnModelMc');

// ── Check 4: wait for MC data to finish loading, badge hides, particle count matches expected ──
await page.waitForFunction(() => (window._FLORIDA_DATA.mc || {}).particles?.length > 0, { timeout: 60000 });
await page.waitForFunction(() => !document.getElementById('mcLoadBadge').classList.contains('visible'), { timeout: 5000 });
const finalCount = await page.evaluate(() => window._FLORIDA_DATA.mc.particles.length);
console.log('CHECK4 final mc particle count (expect 82861, not doubled):', finalCount);

// ── Check 5: filter pills populated after data lands ──
const yearPillCount = await page.locator('#yearPills > span').count();
console.log('CHECK5 year pill count (expect > 1):', yearPillCount);

console.log('--- console/page messages ---');
console.log(msgs.join('\n'));

await browser.close();
```

- [ ] **Step 3: Run it**

Run: `cd /private/tmp/claude-501/-Users-ustropics-Documents-tess-wavetowaste/02664d3c-9afe-424a-9d82-d1b4fa3ebaad/scratchpad && node verify_lazy_load.mjs`

Expected output (order may vary slightly, but all lines must appear with these values):
```
CHECK1 loader hidden early, mc particles at that point: 0
CHECK2 badge visible while loading: true
CHECK4 final mc particle count (expect 82861, not doubled): 82861
CHECK5 year pill count (expect > 1): <some number greater than 1>
--- console/page messages ---
```
(the `--- console/page messages ---` block should NOT contain any `[pageerror]` lines, and should NOT contain `Failed to load data`)

- [ ] **Step 4: If CHECK1 shows `mc particles at that point` is nonzero**

That means the loader hid only after MC data was already loaded (i.e., the background load finished faster than the check ran) — rerun; this check is about the loader not being *blocked* by MC data, not about catching a precise race window. As long as `loader.hidden` reliably appears within ~10s regardless of the 227MB download, the fix is working. If `loader` never gets the `hidden` class within 10s, that's a real failure — recheck Task 7's bootstrap rewrite.

- [ ] **Step 5: If CHECK4 shows a count other than 82861**

A count higher than 82861 means the shared-promise dedup in `startMcLoadInBackground` (Task 5) or the `switchModel('mc')` routing (Task 6) isn't preventing a duplicate `_loadDataset('mc')` call — recheck that `mcLoadPromise` is being reused, not reset, on the click path.

---

### Task 9: Final full-branch check and push

- [ ] **Step 1: Re-run the verification script once more standalone to confirm stability**

Run: `cd /private/tmp/claude-501/-Users-ustropics-Documents-tess-wavetowaste/02664d3c-9afe-424a-9d82-d1b4fa3ebaad/scratchpad && node verify_lazy_load.mjs`
Expected: same output as Task 8 Step 3.

- [ ] **Step 2: Review the full diff**

Run: `cd /Users/ustropics/Documents/tess/wavetowaste && git diff main -- wave_to_waste.html`
Expected: diff limited to the CSS badge block, badge markup, state vars, `showMcBadge`/`hideMcBadge`, `_fitBoundsToParticles`, `_applyMcData`, `startMcLoadInBackground`, the rewritten `switchModel`, and the rewritten bootstrap IIFE — no unrelated changes.

- [ ] **Step 3: Push**

```bash
git push
```
