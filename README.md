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
| ✈️ **Flights** | airplanes.live | every 20s | Live ADS-B aircraft within 250 nm of the map center, rotated to heading. Moving the map refetches. Popup: callsign, type, altitude, speed, heading. |

## Why these sources

The obvious flight API (OpenSky) has locked anonymous access behind OAuth, and the
`adsb.lol` / `adsb.fi` mirrors block browser CORS — **airplanes.live** was the one free,
no-key, CORS-clean ADS-B feed that worked. USGS and wheretheiss.at are rock-solid and
public. Everything runs client-side; there is no server and no key anywhere.

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
  happened, scores itself against a **naive persistence baseline** (predict = last window's
  count), and updates λ. Persists in `localStorage`; backfills on reload from the catalog.
  - **Skill vs naive** = `1 − MAE_model / MAE_naive`. Because seismic *rates* are roughly
    stationary, the learner typically posts **positive skill** — it really does beat naive.
    That's the honest punchline: self-learning works here (real structure) where it didn't for
    crypto prices (no structure).
- **Aftershock watch (Omori).** If an M≥5 mainshock hit in the last 6h, it flags it and notes
  aftershocks decay ∝ 1/t. This is descriptive (a real, known law), not a prediction of the
  next specific quake — **nobody can predict the time/place/size of the next big one, and this
  tool doesn't pretend to.**

## Notes / limits

- **Flights are center-based**, capped at 250 nm and 500 markers — zoom into a region
  (a city, an ocean corridor) for the useful view; a whole-world zoom only shows planes
  near the center point.
- Map tiles are CartoDB "dark" — intentionally muted so the data pops; continents look
  faint by design.
- Be a good citizen: the refresh intervals are deliberately gentle (these are free
  community services). Don't crank them down.
