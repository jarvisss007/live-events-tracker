# Live Events Tracker

A single-file, no-backend, **no-API-key** live world map plotting three real-time
feeds at once: earthquakes, the ISS, and live aircraft. Open it and it connects.

## Run it

- **Double-click `index.html`** — opens in your browser and starts streaming.
- Or serve it: `python3 -m http.server 8788` in this folder → <http://localhost:8788>.

Needs an internet connection (for the map tiles + the data feeds). Toggle each layer
with the checkboxes; counts and a green/red status dot show each feed's health.

## The three feeds (all free, no key)

| Layer | Source | Update | Notes |
|---|---|---|---|
| 🔴 **Earthquakes** | USGS `all_day.geojson` | every 60s | Every quake worldwide, last 24h. Circle size & color = magnitude; popup has place, depth, time-ago. |
| 🛰️ **ISS** | wheretheiss.at | every 5s | Live position + altitude/velocity, a ~2,200 km visibility footprint ring, and a dashed ground-track trail. |
| ✈️ **Flights** | airplanes.live | every 20s | **Unavailable since 2026-09-27 (HTTP 403)** — kept in the UI with that label, not deleted. When it worked: live ADS-B aircraft within 250 nm of the map center, rotated to heading; moving the map refetches. |

## Why these sources

The obvious flight API (OpenSky) has locked anonymous access behind OAuth, and the
`adsb.lol` / `adsb.fi` mirrors block browser CORS — **airplanes.live** was the one free,
no-key, CORS-clean ADS-B feed that worked. USGS and wheretheiss.at are rock-solid and
public. Everything runs client-side; there is no server and no key anywhere.

**Status 2026-09-27:** airplanes.live has since closed anonymous API access (every request
returns 403 "contact us"), so the Flights layer is **down** — its dot shows red and the
footer shows `flights as of —`. adsb.lol / adsb.fi still send no CORS headers. Reviving it
needs either a registered airplanes.live key or a tiny local proxy.

**Freshness:** every seismic section is stamped with the USGS `generated` time of the data
it holds, and turns amber (with "last fetch FAILED, showing old numbers") when a fetch fails
or the data ages past 5 min (all_day) / 11 min (4.5_month). The footer reports per-feed
"as of" times, never a bare clock. Forecast windows that fell off the 24h feed while the
page was closed are dropped, not scored as zero quakes.

## Seismic learning (the self-adapting part)

The top-right **Seismic Learning** panel is a genuine, honest learning module — and a
deliberate contrast to the crypto forecaster. Two of the three feeds (ISS orbit, individual
flight paths) are deterministic physics with nothing to learn; earthquakes are where real
statistical structure lives.

- **Gutenberg–Richter b-value (fits live, adapts).** Aki–Utsu maximum-likelihood fit of the
  magnitude-frequency law on the globally-complete **M4.5+ 30-day** catalog
  (`4.5_month.geojson`). Reads ~**1.0** (nature's value) and re-estimates as new large quakes
  arrive. Also reports the completeness `Mc` and the implied expected M≥5/day (~5–6, realistic).
  - *Why a separate catalog:* `all_day` mixes tiny California quakes (complete to ~M1) with
    global ones (complete only to ~M4.5). Fitting one law across that mixed completeness gives
    a garbage b≈0.4. Fixing `Mc = 4.5` on the M4.5+ feed is the correct, honest fit.
- **Adaptive rate forecast (self-scoring).** Every 15 min it predicts the number of quakes in
  the next window (an EWMA-adapted Poisson rate λ), then 15 min later counts what actually
  happened, scores itself against two simple baselines, and updates λ. Baselines: **last-window
  persistence** (predict = last window's count) and a **trailing-24h mean** (the mean 15-min count
  over the 24h the feed still holds; needs ≥12h). Persists in `localStorage`; a prediction that
  matured while the page was closed is scored on reload if it is still inside the 24h feed,
  otherwise it is dropped.
  - **Skill** = `1 − MAE_model / MAE_baseline`, shown against each baseline (the 24h-mean skill is
    computed on exactly the windows that had that baseline).
  - **How to read it:** beating last-window persistence is *not* evidence that seismic rates are
    forecastable. A smoothed mean beats persistence on any stationary count series, because
    persistence's errors carry roughly twice the variance of a smoothed mean's, so positive skill
    against it mostly shows a weak baseline, not structure in the data. The 24h-mean comparison is
    the stricter check; on a stationary series it should land at or slightly below zero (one
    simulated run of stationary Poisson counts, 60 windows, gave +0.14 against persistence and
    −0.04 against the 24h mean). This panel makes no claim that earthquake rates are predictable.
  - **Small n, stored only in your browser.** One window resolves per 15 min, so n stays small, and
    the verdict flips on as few as 4 windows at its ±0.05 threshold. Scores live only in this
    browser's `localStorage` (key `let_seismic_v1`); nothing is stored or sent anywhere else, there
    is no server, and clearing site data resets them.
  - **Counts mix catalogs.** `all_day` combines regional networks (complete to ~M1) with global
    coverage (complete to ~M4.5), so window counts are dominated by a few dense regional networks
    and say nothing about a single, uniformly complete catalog.
- **Aftershock watch (Omori).** If an M≥5 mainshock hit in the last 6h, it flags it and notes
  aftershocks decay ∝ 1/t. This is descriptive (a real, known law), not a prediction of the
  next specific quake — **nobody can predict the time/place/size of the next big one, and this
  tool doesn't pretend to.**

## Notes / limits

- **Flights are center-based**, capped at 250 nm and 500 markers — zoom into a region
  (a city, an ocean corridor) for the useful view; a whole-world zoom only shows planes
  near the center point.
- Map tiles are Esri World Dark Gray Canvas (CARTO dark_all went key-only in 2026 and
  serves an "API KEY REQUIRED" placeholder with HTTP 200) — intentionally muted so the data pops; continents look
  faint by design.
- Be a good citizen: the refresh intervals are deliberately gentle (these are free
  community services). Don't crank them down.
