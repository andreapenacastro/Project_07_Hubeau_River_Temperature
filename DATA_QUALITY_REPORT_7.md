# Data-Quality Report — River water temperature, Ille-et-Vilaine 2023 (Hub'Eau)

**Group:** Abhy, Andrea, Kaled, Sofyan · **Date:** 2026-11-05 · **Notebook:** `project_07_hubeau_river_temperature.ipynb` · **Commit:** `<git short hash>`
**Clean-data fingerprint:** `2dae9d1dcaa3`

---

## 1. Source & provenance
| | |
|---|---|
| Business question | How does river water temperature vary by season and between the stations of one département? Which effects (season, station) must a temperature model include? |
| Source(s) & endpoint / URL | Hub'Eau, API *Température des cours d'eau*: `https://hubeau.eaufrance.fr/api/v1/temperature/chronique` (station list: `/temperature/station`) |
| Collection method (API / scraping / file) & date range | REST API, paginated (`next` link), cached 1 week, retry on 429/5xx, 0.5 s pause. Collected **one station × one month at a time** to stay under the ~20,000-row paging cap. Period 2023-01-01 → 2023-12-31; collected 2026-11-05 20:52 UTC |
| Licence, robots.txt / ToS constraints | Licence Ouverte / Open Licence 2.0 (Etalab), open access, attribution "Source: Hub'Eau". No published rate limit; we throttle and cache. Paging stops silently at ~20,000 rows per query |
| Unit of observation (one row = …) | One hourly temperature reading (°C) at one station, one date and local time. (station, date, hour) is unique except 7 pairs at the spring clock change |
| Final size (rows × columns) | Raw: 51,425 × 23 · Clean (`clean.parquet`): 51,425 × 18 · Daily summary: 2,140 × 11 (8 stations) |

**Why 2023:** 2025 has only 4 complete stations. 2023 has 8 stations with real issues: 4 sensors stop mid-summer, a station replacement (04207400 Flume → 04212700 Semnon, confirmed in the 2025 data), and duplicate timestamps.

## 2. Schema (the contract)
Raw schema: Section 3.4 (`schema`, `strict=True`, validated with `lazy=True`). Key checks:
- `code_station` — matches `^\d{8}$`, kept as text — protects against lost leading zeros / wrong IDs
- `date_mesure_temp` — ISO format, inside the query window — protects against paging or filter errors
- `heure_mesure_temp` — `HH:MM:SS` — protects against malformed times
- `resultat` — in [−1, 35] °C — protects against sensor errors and readings out of the water
- `symbole_unite` — isin `["°C"]` — protects against mixed units
- `longitude` / `latitude` — inside Ille-et-Vilaine — protects against stations from another département

Validation on raw data: **0 violations**. Uniqueness of (station, date, hour) is checked separately (Section 3.4, 4.1): **14 rows / 7 station-hours** repeat, all on 26 March 03:00. Clean schema (final section, 18 columns, `strict=True`, adds `ts`, `timestamp_collision`, `spike` flags; `resultat` now in [0, 30] °C): **passes**.

## 3. Missingness
No empty cells in any column. Missing data = **missing hourly timestamps**: 18,662 of 70,080 expected station-hours (26.6 %), always blocks at the start or end of a series (Section 4.3 heatmap and season table).

| Column / subset | % missing | Mechanism | Evidence | Decision |
|---|---|---|---|---|
| `resultat` (cells) | 0 % | — | overview table, 3.3 | none needed |
| `ts` — Aff, Aron, Chênelais | 0.0 % | — | only the 26 March 03:00 label repeats | none |
| `ts` — Couesnon (stops 29 Jul), Flume (stops 30 Jul) | 42.6 %, 42.4 % | MAR (station × date) | permanent stops; Flume stops the day Semnon starts | not imputed |
| `ts` — Vilaine, Illet (stop 25 Aug) | 35.2 % each | MAR; MNAR possible for Illet | stop 1 h apart = operator decision; Illet's last 5 days swing 3.9 °C vs 2.6 °C usual (others 1.6 °C) | not imputed; Illet end flagged |
| `ts` — Semnon (starts 30 Jul) | 57.7 % | MAR | new station replacing Flume | not imputed |

## 4. Duplicates & outliers — decision table
From the `decision_table` (Section 4.5).

| Column | Issue | n rows | Decision | Why |
|---|---|---|---|---|
| geometry_coordinates | GeoJSON coordinate lists | 51,425 | split into longitude/latitude | lists are not hashable; numeric columns keep the location |
| all | exact duplicate rows | 0 | dropped | identical on every column |
| ts | same station + timestamp twice | 14 | flagged; both readings kept | API has no timezone offset; clock-change cause unverified |
| code_unite | unit other than °C | 0 | dropped (none found) | mixed units would corrupt every average |
| resultat | outside [0, 30] °C | 0 | set to NaN (none found) | below 0 = ice; above 30 implausible in Brittany |
| code_qualification | not validated ("Non qualifié") | 51,425 | kept, flagged as limitation | no producer check, so our own checks matter |
| ts | timestamps absent vs 24/day | 18,662 | not imputed | block gaps, MAR |
| resultat | spikes: > 2 °C in 1 h or > 5 °C from 7-day median | 0 | set to NaN (none found) | too fast for a river: sensor out of water |
| ts | spike tests skipped at collisions | 14 | not evaluated; kept | no unique timestamp to compare |
| resultat | deviations > 3 °C, below spike thresholds | 115 | kept | gradual real events (June heat, 17 Oct cold drop) |
| ts | repeated slots in daily summaries | 7 | averaged inside daily summary only | each hour label gets equal weight |
| ts | incomplete days | 5 | dropped from daily table (< 18 h) | a daily mean needs ≥ 18 of 24 hours |

## 5. Key findings (max 3)
1. **The season sets the level** — 2.2 → 24.7 °C, median 12.6 °C — evidence: Fig. 5_1 (ECDF) and Fig. 5_2 (monthly profiles of the 3 full-year stations, Aff, Aron, Chênelais) — so what: any model needs a month or season effect.
2. **Stations differ strongly on the same day** — on 25 June the Vilaine averaged 23.4 °C and the Aff 16.8 °C (6.6 °C gap); the Illet swings ~2.6 °C day/night vs ~1 °C elsewhere — evidence: Fig. 5_3 (station × month heatmap), Section 4.4 triage table — so what: add a station effect; consider daily amplitude per station.
3. **Station coverage changes with the season** — spring has 7 stations, autumn only 4 — evidence: Section 4.3 heatmap and season table; blank cells in Fig. 5_3 — so what: compare seasons on the full-year stations (as Fig. 5_2 does), never on pooled averages.

## 6. What cannot be trusted ⚠️
- **All of `resultat`**: every row is `code_qualification = 4` ("Non qualifié"), raw sensor data never validated by the producer.
- **Pooled autumn statistics**: only Aff, Aron, Chênelais and Semnon cover Sep–Dec, so an "autumn average" describes different rivers than the summer one. This also affects per-station medians over the year (Fig. 5_3 title): stations that stop in summer have no autumn/winter end-of-year values.
- **Illet at Ercé-près-Liffré, last days before 25 Aug 2023**: day/night swing much larger than usual; possibly a sensor in low water reading partly air.
- **26 March 2023, 03:00, all 7 stations**: two readings per label and no timezone in the API, so their order and true time are unknown.
- **Flume (04207400) + Semnon (04212700)**: a replacement, but on two different rivers; never join them into one series.
- **Generalisation**: one département, one year; do not extend results to other regions or years.

## 7. Hypotheses for modelling (≥ 5)
- **H1** — Daily mean temperature is higher in summer (Jun–Aug) than in winter (Dec–Feb) at the 3 full-year stations — one-sided Mann-Whitney U per station.
- **H2** — Daily mean temperature differs between stations over a common period (1 Jun–29 Jul, 7 stations) — Kruskal-Wallis, then Dunn post-hoc.
- **H3** — Daily amplitude (max − min) is larger at the Illet than at the other stations in summer — Mann-Whitney U.
- **H4** — Daily amplitude increases with daily mean temperature, within each station — Spearman correlation (ρ > 0).
- **H5** — Full-year stations move together: their daily means are strongly correlated (ρ > 0.9) across 2023 — Spearman on paired days.
- **H6** — The Illet's amplitude in its last 5 days is higher than in the rest of its summer — Mann-Whitney U (tests the MNAR doubt).

## 8. How to reproduce
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute project_07_hubeau_river_temperature.ipynb
```
Expected fingerprint: `2dae9d1dcaa3` (stable while the API cache is valid; without the cache Hub'Eau data may have changed).
