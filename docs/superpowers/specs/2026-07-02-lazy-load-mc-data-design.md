# Lazy-load MC dataset in wave_to_waste.html

## Problem

`wave_to_waste.html` bootstrap currently awaits the full MC dataset
(`florida_mc_data_1/2/3.js`, ~227MB combined) before rendering the map or
enabling any UI. Since no years/months are selected by default, this
227MB download blocks a page that would otherwise show nothing anyway.

## Goal

Render map + UI immediately after the small nesting-zones file loads.
Load the MC dataset in the background afterward, with a lightweight
non-blocking indicator, and apply any filter/model choices the user made
while it was still loading.

## Design

### Bootstrap sequence

1. `_loadScript('florida_nesting_zones.js')` — unchanged, still awaited
   (small file, fast).
2. `buildNestingIndex()` — unchanged.
3. Call `init()` immediately with `allParticles = []`. Map, basemap,
   filter UI, model buttons all render and are interactive right away.
4. Kick off `_loadDataset('mc')` via `.then()/.catch()`, not awaited by
   bootstrap.

### Loading indicator

- Full-screen blocking `#loader` is no longer shown on initial page load.
- A small corner badge ("Loading MC data…") shows while the background
  fetch is in flight.
- On success: badge hides, `allParticles` populated, any queued filter
  state applied, layers rebuilt, map bounds fit.
- On failure: badge text switches to the existing "Failed to load
  data…" message and stays visible (non-blocking — rest of the page,
  including nesting-zone toggle and basemap switching, stays usable).

### Queueing early interaction

- New flag `mcDataReady` (bool) plus `pendingFilterState` object
  capturing whatever filter pills / model selection the user made while
  `mcDataReady` is false.
- Filter-pill clicks and model-switch calls still update UI state
  immediately (so the controls feel responsive) but skip any
  particle-dependent work (`rebuildLayers`, `fitBounds`) until
  `mcDataReady`.
- When the background load resolves, apply the queued state in one pass:
  `rebuildFilterPills(); rebuildLayers();` etc., then fit bounds.
- Only affects the initial `mc` load path. `det`/`stoch` are unaffected —
  they already lazy-load on demand via `switchModel`, which already
  blocks on its own dataset load (that path is fine, small window).

### Non-goals

- No change to `_loadScript`/`_loadDataset` internals.
- No change to `det`/`stoch` on-demand loading behavior.
- No service worker / caching changes.

## Testing

- Headless browser load: confirm map renders and is interactive before
  MC particle count is nonzero.
- Confirm badge appears then disappears once `window._FLORIDA_DATA.mc`
  particle count is populated.
- Confirm clicking a filter pill or model button before data is ready
  gets applied automatically once data lands (no dead click, no
  duplicate application).
- Confirm failure path (simulate by renaming a data file) shows badge
  error text, not full-page block.
