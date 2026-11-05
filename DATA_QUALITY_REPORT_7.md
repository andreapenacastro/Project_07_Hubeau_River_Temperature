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
1. **Season is the biggest effect.** The 3 full-year stations (Aff, Aron, Chênelais) share one cycle: coldest in February (6.3–7.4 °C), warmest June–September (16–19 °C), a 9–12 °C swing. The ECDF is one smooth curve from 2.2 to 24.7 °C: that spread is the seasons, not bad values — evidence: Fig. 5.1, 5.2 — so what: any model needs a month or day-of-year term.
2. **Stations are close in winter, far apart in summer.** On the same days in Jan–Feb the 7 live stations are within 0.9 °C; from 1 Jun to 28 Jul the gap is 5.4 °C (Vilaine 21.0 °C vs Aff 15.7 °C) — evidence: Fig. 5.3 — so what: include a station effect **and** a station × season interaction.
3. **The stations that drop out after summer are the warm ones.** Couesnon, Flume, Illet and Vilaine stop 29 Jul–25 Aug; in Jun–Jul they average 19.1 °C vs 16.7 °C for the 3 full-year stations — evidence: 4.3 heatmap, blank cells in Fig. 5.3 — so what: compare stations only on dates where all are live; no imputation for autumn.

## 6. What cannot be trusted ⚠️
- **Fig. 5.2 as "the département's seasonal cycle"**: the 3 full-year stations are 2.4 °C cooler in summer than the 4 that stopped, so the département's summer peak is underestimated.
- **Any yearly average per station (incl. the 2.6 °C in Fig. 5.3's title)**: 4 stations have no autumn and Semnon only Aug–Dec; Semnon and Vilaine look warmest partly because they miss cold months.
- **The pooled ECDF and its 12.6 °C median (Fig. 5.1)**: the 3 full-year stations are 51 % of the rows, Semnon 7 %, and Jan–Jul outweighs Sep–Dec. It describes our rows, not the typical river.
- **`resultat`, all 51,425 rows**: `code_qualification = 4` ("Non qualifié"). Our checks catch fast errors only; a slow drift or calibration offset would pass.
- **Illet at Ercé-près-Liffré, last days before 25 Aug**: day/night swing 3.9 °C vs 2.6 °C usually (others 1.6 °C) — possible low water, unconfirmed.
- **26 March 2023, 02:00–03:00, 7 stations**: 14 rows share the 03:00 label and 02:00 is missing; probably the clock change, unverifiable (no timezone). 7 hours of ~51k.
- **Flume (04207400) + Semnon (04212700)**: a replacement on two different rivers; never join them into one series.
- **Generalisation**: 1 département, 1 year, 8 point sensors; says nothing about other regions, years or trends.

## 7. Hypotheses for modelling (≥ 5)
- **H1** — daily `temp_mean` is higher in summer (Jun–Aug) than winter (Dec–Feb) · 3 full-year stations · one-sided Mann-Whitney U per station, on weekly means.
- **H2** — daily `temp_mean` is higher at Vilaine than at Aff on the same day · 1 Jun–28 Jul 2023 (58 days) · paired Wilcoxon signed-rank.
- **H3** — the gap between stations is larger in summer than winter (0.9 vs 5.4 °C) · 7 stations live 1 Jan–28 Jul · OLS `temp_mean ~ C(station) * C(season)` on weekly means, F-test on the interaction.
- **H4** — the daily swing `temp_max − temp_min` is larger in summer than winter (median 1.6 vs 0.9 °C) · stations with both seasons · Kruskal-Wallis across seasons, then Mann-Whitney per station.
- **H5** — daily `temp_mean` is strongly autocorrelated (lag-1 > 0.9; we see 0.97–1.00) · every station · ACF + Ljung-Box. If it holds, H1–H4 must use weekly blocks or AR errors, or p-values look too good.

**Next steps:** collect 2019–2024 with the same loop (are summer peaks rising?); add air temperature and river flow (Hub'Eau `hydrometrie`); check whether validated 2023 data exists.

## 8. How to reproduce
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute project_07_hubeau_river_temperature.ipynb
```
Expected fingerprint: `2dae9d1dcaa3` (stable while the API cache is valid; without the cache Hub'Eau data may have changed).
