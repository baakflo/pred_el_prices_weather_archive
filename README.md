# pred_el_prices weather archive

Forward-collected data archive and production state store for
[pred_el_prices](https://github.com/baakflo/pred_el_prices) — a daily
day-ahead electricity price forecast for DE-LU
([live site](https://predict.baakes-systems-modeling.eu)). Written daily by
unattended GitHub Actions workflows in that repo.

Why forward-collected: the upstream sources serve only a rolling window or
only the latest version — DWD deletes open-data files after ~24 h,
ECMWF open data is a rolling window, ENTSO-E overwrites forecasts with the
latest submitted version, PEGELONLINE keeps ~31 days. For anything before
its collection start date, **this archive is the only history that exists**,
and days missed are unrecoverable. That is also why the layout is one
immutable file per day: nothing here is ever rewritten.

## Datasets

### `icon-eu-eps/` — DWD ICON-EU-EPS ensemble (daily 00Z run)

```
icon-eu-eps/<YYYY>/icon-eu-eps_<YYYYMMDD>00.parquet   # one file per daily 00 UTC run
```

Per-member regional aggregates over Germany. One row per
(cell, member, variable, valid hour); ~554k rows / ~3 MB per day.

| column | meaning |
|---|---|
| `cell_lat`, `cell_lon` | lower-left corner of a 1° x 1° cell; box 47-56°N, 5-16°E (99 cells) |
| `member` | ensemble member 0-39 |
| `variable` | `u_10m`, `v_10m` (10 m wind, m/s), `aswdir_s`, `aswdifd_s` (direct/diffuse shortwave down, W/m²), `t_2m` (2 m temperature, K) |
| `valid_time` | forecast valid time (UTC); steps +21h to +48h of the run |
| `run_time` | model run initialization (00 UTC) |
| `value` | mean over the ICON-EU grid points (~13 km) inside the cell, per member |

Notes:

- Radiation fields follow the ICON convention: **averages since model start**, not
  instantaneous fluxes. Hourly flux over (t1, t2): `(avg2*t2 - avg1*t1)/(t2 - t1)`.
- No hub-height wind exists in the EPS open data; extrapolate from 10 m.
- The cell box overlaps neighbouring countries slightly; weight cells downstream.

### `ecmwf-ens/` — ECMWF IFS ENS ensemble (00Z, plus 12Z fallback vintage)

```
ecmwf-ens/<YYYY>/ecmwf-ens_<YYYYMMDD><HH>.parquet   # HH = 00 or 12 (run hour, UTC)
```

Same schema and 1° cell box as `icon-eu-eps`, from ECMWF open data (0.25°).
Differences:

- `member`: 0-50 (control run stored as member 0).
- `variable`: `u_10m`, `v_10m`, `t_2m`, `u_100m`, `v_100m` (100 m wind — the
  closest open-data field to hub height), `ssrd` (surface shortwave down;
  **accumulated since model start**, J/m² — de-accumulate for hourly flux).
- Steps are 3-hourly: the 00Z run covers +21h..+48h (the next delivery day);
  the 12Z run covers +33h..+60h and is archived each evening as the
  **fallback vintage** — if the next morning's 00Z download fails, the
  production forecast runs on this file instead of missing the day.

### `entsoe-forecasts/` — pre-auction ENTSO-E forecast vintages

```
entsoe-forecasts/<YYYY>/entsoe-forecasts_<YYYYMMDD>.parquet   # dated by DELIVERY day
```

Day-ahead **load forecast** and aggregated **wind/solar forecast** for DE-LU,
snapshotted the evening before the delivery day (the TSO day-ahead RES
forecast publishes at 18:00 local). ENTSO-E serves only the latest submitted
version of these series, so the vintage a forecaster actually saw around
auction time must be captured forward — that is what these files are.

### `pegel-kaub/` — Rhine water level at Kaub

```
pegel-kaub/<YYYY>/pegel-kaub_<YYYYMMDD>.parquet   # one file per day
```

Raw PEGELONLINE measurement timeseries for the Kaub gauge, the bottleneck
for fuel-barge logistics on the Rhine: low water restricts coal/oil barge
loads and shows up in marginal costs. Upstream keeps ~31 days of raw
measurements, so the daily job also self-heals up to a month of gaps.

### `site-state/` — production state for the daily forecast

Not a curated dataset but the live pipeline's state store, versioned here so
every published forecast is auditable:

```
site-state/
  cache/entsoe/...                 # local cache of ENTSO-E series (prices, load, RES forecasts)
  cache/energy_charts/...          # monthly installed wind/solar capacity
  ens_features.parquet             # daily ENS weather feature table
  site/latest.json                 # what the website serves right now
  site/history.json                # per-day curves: forecast vs actual
  site/forecast_log.parquet        # append-only pre-gate forecast log — the scorecard's source of truth
```

The forecast log is written before the 12:00 CET auction gate and never
retouched; entries carry honesty flags (`pre_gate`, weather vintage). Losing
it would break the public scorecard, which is why it lives in a git history.

## Licenses / attribution

Each dataset redistributes third-party open data under its source's terms:

- **`icon-eu-eps/`** — contains data from Deutscher Wetterdienst (DWD) open
  data, processed. Source: Deutscher Wetterdienst,
  [opendata.dwd.de](https://opendata.dwd.de) (GeoNutzV / CC BY 4.0-equivalent
  attribution requirements).
- **`ecmwf-ens/`** — contains modified ECMWF open data © ECMWF
  ([open data license](https://www.ecmwf.int/en/forecasts/datasets/open-data),
  CC BY 4.0). Attribution: "Contains modified ECMWF forecast data".
- **`entsoe-forecasts/`, `site-state/cache/entsoe/`** — source: ENTSO-E
  Transparency Platform ([transparency.entsoe.eu](https://transparency.entsoe.eu));
  reuse with source attribution.
- **`pegel-kaub/`** — source: Wasserstraßen- und Schifffahrtsverwaltung des
  Bundes (WSV), [PEGELONLINE](https://www.pegelonline.wsv.de), licensed
  [DL-DE-BY-2.0](https://www.govdata.de/dl-de/by-2-0).
- **`site-state/cache/energy_charts/`** — source:
  [energy-charts.info](https://energy-charts.info) (Fraunhofer ISE), CC BY 4.0.
- **Project-generated artifacts** (`site-state/site/`, `ens_features.parquet`)
  — © the pred_el_prices project; reuse freely with attribution and a link
  to the [live site](https://predict.baakes-systems-modeling.eu).

No commercial or non-redistributable data lives in this repo: fuel-price
proxies (Yahoo) are rebuilt locally by anyone reproducing the results, and
the energyforecast.de benchmark snapshots are collected in a private repo.
