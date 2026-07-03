# Flight Corridor Advisory System

An end-to-end decision-support prototype that forecasts turbulence risk from
real pilot weather reports and recommends safer route deviations — combining
a live data pipeline, an LSTM forecasting model, and rule-based agent logic.

**This is a decision-support prototype, not an autonomous flight planner.**
Any real route change must be requested from and approved by ATC.

## What it does

1. **Collects real data** — pulls live PIREPs (Pilot Reports) from
   [AviationWeather.gov](https://aviationweather.gov)'s public API, covering
   turbulence and icing reports across the continental US.
2. **Automates collection** — an [n8n](https://n8n.io) workflow runs every
   30 minutes, fetches new reports, scores turbulence severity, and writes
   to a live [Supabase](https://supabase.com) (Postgres) database. It also
   posts a Slack alert when a report crosses a severity threshold.
3. **Forecasts risk** — an LSTM (trained in `notebooks/02_lstm_training.ipynb`)
   learns from an hourly turbulence-risk time series per US region and
   predicts near-term risk, benchmarked against a naive persistence baseline.
4. **Recommends routes** — the agent (`notebooks/03_agent.ipynb`) generates
   candidate route deviations for a given flight, blends the LSTM's regional
   forecast with a distance-weighted spatial estimate from nearby live
   reports, and ranks candidates by predicted risk vs. added fuel/time cost.
5. **Visualizes it live** — `dashboard/index.html` is a standalone browser
   dashboard: a live map of recent reports, per-region risk bars, and an
   interactive route advisory tool.

## Architecture

```
AviationWeather.gov (PIREP API)
        │
        ▼
     n8n  ──────────────► Slack (severe turbulence alerts)
        │
        ▼
   Supabase (Postgres)
        │
        ▼
  LSTM (Python/Keras)  ──► trained in Colab, forecasts next-hour
        │                   regional turbulence risk
        ▼
  Routing agent  ──────► ranks candidate deviations by risk vs. cost
        │
        ▼
     Dashboard  ────────► live browser view + recommendation
```

## Repo structure

```
notebooks/
  01_data_pipeline.ipynb   — pulls, cleans, and stores PIREP/METAR data
  02_lstm_training.ipynb   — builds the hourly risk grid, trains the LSTM
  03_agent.ipynb           — route generation, risk blending, ranking

dashboard/
  index.html               — standalone interactive dashboard

n8n/
  workflow_export.json     — the automated collection + alerting workflow
```

## Setup

1. **AviationWeather.gov** — no API key needed, just a custom User-Agent header.
2. **Supabase** — create a free project, run the SQL in `notebooks/01_data_pipeline.ipynb`
   (Step 2) to create the `pireps` table, and add Row Level Security policies
   allowing `anon` insert/select (see notebook for the exact SQL).
3. **n8n** — import `n8n/workflow_export.json`, add your Supabase and Slack
   credentials in the relevant nodes.
4. **Colab** — open the notebooks in order, fill in your Supabase URL/key
   at the top of each, run top to bottom.
5. **Dashboard** — open `dashboard/index.html` in any browser, add your
   Supabase credentials in the `<script>` section, or leave as-is to run
   in demo mode with sample data.

## Notable engineering decisions

- **API bounding box order.** AviationWeather.gov expects
  `minLat,minLon,maxLat,maxLon` — not the more common GeoJSON order. Getting
  it backwards doesn't error, it silently returns zero results.
- **Row Level Security.** Supabase blocks all inserts by default; explicit
  `anon` policies were required for both reads and writes.
- **n8n's implicit array-splitting.** An HTTP Request node automatically
  splits a JSON array response into one item per element for downstream
  nodes — code assuming a raw array silently dropped every record until
  this was traced through execution logs.
- **Coarse-grid limitation, solved with blending.** Six large US regions
  were too coarse to differentiate nearby route deviations. Rather than
  subdividing into smaller regions (which would starve each of training
  data), the agent blends the LSTM's regional forecast with a
  distance-weighted spatial interpolation of actual nearby reports.
- **Honest evaluation.** The LSTM is benchmarked against a naive
  persistence baseline ("assume next hour = this hour"), not evaluated in
  isolation.

## Known limitations

- Dataset size is still growing via the live n8n pipeline; forecast quality
  will improve with more collected history.
- The 6-region grid is a deliberate simplification; a production version
  would use a finer spatial grid.
- The dashboard's live route advisory uses a client-side spatial
  approximation, not the trained Keras model directly (browsers can't run
  it) — the notebooks contain the full model-based evaluation.

## Tech stack

Python · TensorFlow/Keras · pandas · scikit-learn · n8n · Supabase (Postgres) ·
AviationWeather.gov API · HTML/CSS/JS
