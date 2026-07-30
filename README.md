# pred_el_prices weather archive

Forward-collected archive of DWD **ICON-EU-EPS** ensemble forecasts over Germany,
distilled to per-member regional aggregates. Written daily by a GitHub Actions
workflow in [pred_el_prices](https://github.com/baakflo/pred_el_prices); DWD's open-data
server keeps only ~24 h of files, so this archive is the only history that exists.

## Layout

```
icon-eu-eps/<YYYY>/icon-eu-eps_<YYYYMMDD>00.parquet   # one file per daily 00 UTC run
```

## Schema

One row per (cell, member, variable, valid hour). ~554k rows / ~3 MB per day.

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

## License / attribution

Contains data from Deutscher Wetterdienst (DWD) open data, processed.
Source: Deutscher Wetterdienst, [opendata.dwd.de](https://opendata.dwd.de)
(GeoNutzV / CC BY 4.0-equivalent attribution requirements).
