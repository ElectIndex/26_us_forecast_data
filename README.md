# 26_us_forecast_data

Data for ElectIndex's **2026 U.S. election forecast** (Governor, Senate, House) —
curated model inputs plus the generated forecast output.

## Inputs

| File | Description |
|------|-------------|
| `candidates.csv` | Curated candidate database (name, party, status) per race. |
| `races.csv` | Race definitions — chamber, seat, incumbency. |
| `demographics.csv` | District / state demographic priors. |
| `county_segments.csv` | 2024 CD×county vote segments used for county projection. |
| `historical.csv` | Historical results behind candidate strength and partisan lean. |
| `national.csv` | National-level inputs. |
| `national_econ.csv` | Economic training set (fundamentals). |
| `gcb_polls.csv` | Generic-ballot polls (538-format, pollster-quality rated). |
| `approval_polls.csv` | Presidential approval polls. |
| `pollster_ratings.csv` | Pollster quality ratings. |
| `third_parties.csv` | Third-party candidates detected from 2026 FEC filings. |
| `fred_cache.json` | Cached FRED economic series (value + fetch timestamp). |

## Data dictionary

Column-level reference for the input files. Conventions: margins are Democratic
minus Republican in percentage points (negative = Republican advantage); `VAP` =
voting-age population; candidate-slot IDs (`*_id`) join to `candidates.csv`; and
geography keys (`race_code`, district codes, `county_fips`) are shared across
files.

### candidates.csv — 1,012 rows (one per candidate)
- `cand_id` — internal candidate ID (e.g. `C-0001`), referenced by `races.csv`.
- `name`, `first`, `last` — candidate name.
- `party` — `DEM` / `REP` / `IND` / minor party.
- `state` — two-letter state.
- `incumbent` — `1` if the incumbent, else `0`.
- `fec_id` — FEC candidate ID (blank if none).
- `strength` — candidate-strength score (electoral over/under-performance vs. the
  seat's partisan baseline; restored by name-match from historical results, `0`
  for candidates with no history).

### races.csv — 506 rows (one per contest)
- `race_code` — race key (e.g. `AK-SEN`, `AK-01`).
- `race_type` — `senate` / `governor` / `house`.
- `state`, `district` — geography.
- `dem_id`/`dem_name`/`dem_party`, `rep_*`, `ind_*` — candidate in each slot
  (`*_id` joins to `candidates.csv`).
- `ind_caucus` — party an independent caucuses with.
- `dem2_id`, `rep2_id` — second same-party candidate (top-two same-party races).
- `rcv` — ranked-choice race (bool).
- `threeway` — modeled as a 3-way D/R/IND plurality (bool).
- `ind_frac` — share of the anti-R vote going to the independent in 3-way sims.
- `same_party` — `DEM`/`REP` when a California top-two general is same-party.
- `cand2_name` — second candidate in a same-party / intra-party contest.
- `cand1_inc` — candidate 1 is the incumbent (bool).
- `incumbent_party`, `seat_holder` — party currently holding the seat, and who.
- `dem_status`/`rep_status`/`ind_status` — `confirmed` / `presumptive` /
  `projected`.

### demographics.csv — 435 rows (one per House district)
Voting-age population broken down four ways:
- `district_code`, `state`, `district`, `vap_total`.
- `vap_white`/`vap_hispanic`/`vap_black`/`vap_asian`/`vap_native`/`vap_pacific` —
  VAP by race/ethnicity.
- `age_20_29` … `age_80_plus` — VAP by age band.
- `educ_no_hs` … `educ_doctorate` — VAP by educational attainment.
- `inc_u10k` … `inc_200k_plus` — counts by income bracket.

### county_segments.csv — 3,744 rows (CD×county intersections)
2024 presidential votes and demographics for each (county, congressional
district) segment; the basis for uniform-swing county projection.
- `state`, `county_fips`, `cd_code`.
- `votes_dem`/`votes_rep`/`votes_total` — 2024 presidential votes in the segment.
- `pop_tot`, `vap_tot`, and `vap_white`/`vap_black`/`vap_hisp`/`vap_asian`/
  `vap_native`/`vap_pacific`.

### historical.csv — 485 rows (district + state level)
Past presidential results behind partisan lean and candidate strength.
- `code`, `level` (`district`/`state`), `state`, `district`.
- `pres16_total`/`pres16_dem`/`pres16_rep` (and `pres20_*`, `pres24_*`) — votes.
- `margin_2016`/`margin_2020`/`margin_2024` — D−R margin in points.

### national.csv — 3 rows
National two-party result by recent year: `Year`, `Margin` (D−R points).

### national_econ.csv — 33 rows (one per presidential election, 1960→)
Economic-fundamentals training set.
- `Year`, `Incumbent` (party in the White House), `Winner`, `Margin`
  (popular-vote margin).
- `Jobs`, `Income`, `Consumption`, `Production`, `Inflation`, `GDP`, `Stocks` —
  economic indicators feeding the fundamentals model.

### gcb_polls.csv — 496 rows
Generic-congressional-ballot polls.
- `pollster`, `end_date`, `dem`, `rep` (topline %), `n` (sample size),
  `rating` (pollster quality), `lean` (partisan-sponsor flag).

### approval_polls.csv — 878 rows
Presidential approval polls.
- `pollster`, `end_date`, `approve`, `disapprove` (%), `n`, `rating`, `lean`.

### pollster_ratings.csv — 73 rows
Pollster-quality lookup: `pollster`, `rating` (higher = better), `lean`
(house-effect direction; blank if none).

### third_parties.csv — 99 rows
Third-party / independent candidates on the 2026 ballot, detected from FEC
filings: `race_code`, `party`, `name`, `fec_id`.

## Output

Generated forecast output lives in [`output/`](output/). Each run updates the
per-race timelines and the latest-run aggregates, and archives a dated snapshot
under `output/historical/`.

### Per-race timelines

- `output/races/<RACE_CODE>.csv` — **one row per run date**, tracking the race's
  evolution over time: `margin`, `dem_prob`/`rep_prob`, `rating`, `lean`, `gcb`,
  `fundamentals`, `pres_2024`, `swing_24_26`, vote shares, RCV / three-way /
  same-party flags, independent + third-party fields, localized Trump approval
  (`environment`, `trump_approve`, `trump_net`), and the Dem two-party share for
  every demographic group (`dem_vap_white`, `dem_age_20_29`, `dem_educ_bachelor`,
  `dem_inc_200k_plus`, …).

### Latest-run aggregates (overwritten each run)

- `output/races_summary.csv` — one row per race.
- `output/chambers.csv` — per-chamber simulated seat distribution and control
  probabilities.
- `output/national_indicators.csv` — generic ballot (naive vs corrected),
  approval, fundamentals.
- `output/area_indicators.csv` — per House district and state: lean, GCB,
  environment, and localized Trump approval.
- `output/county_projections.csv` — county-level D/R/IND/OTH vote projection
  (latest run only).
- `output/county_approval.csv` — Trump approval per county (latest run only).
- `output/rcv_rounds.csv` — round-by-round elimination for the six RCV races.

### Timelines & history

- `output/timeline.csv` — daily national + chamber numbers across the cycle.
- `output/historical/<YYYY-MM-DD>/` — per-date archive of that run's run-level
  aggregates: `races_summary.csv`, `chambers.csv`, `national_indicators.csv`,
  `area_indicators.csv`, `rcv_rounds.csv`. County-level files are not archived
  per date; per-race history lives in the `races/` timelines.

Produced by the ElectIndex 2026 U.S. forecast pipeline. Inputs are curated from
public sources; the per-race timelines and `historical/` snapshots accumulate
over time, while the top-level aggregates reflect the most recent run.
