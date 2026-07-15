# 26_us_forecast_data

Data for ElectIndex's **2026 U.S. election forecast** (Governor, Senate, House, and
state legislatures) — curated model inputs plus the generated forecast output. The
federal/governor forecast and the state-legislative forecast share this repo; files
prefixed `leg_` belong to the state-legislative layer (`src/pipeline/run_stateleg.py`),
the rest to the federal/governor pipeline (`src/pipeline/run_all.py`).

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
| `race_polls.csv` | Per-race NYT matchup polls (Senate/House/Governor), multi-candidate. |
| `pollster_ratings.csv` | Pollster quality ratings; the `banned` column (rating 0) denotes fabrication/fake pollsters excluded from every average. |
| `third_parties.csv` | Third-party candidates detected from 2026 FEC filings. |
| `fundraising.csv` | Per-race candidate fundraising (cycle receipts + cash on hand): House/Senate from the FEC bulk summary; governors from Transparency USA state campaign-finance data (21 of 36 races — uncovered states 0). Feeds the fundamentals fundraising term and the output `dem_funds`/`rep_funds`/`ind_funds`. |
| `candidate_meta.csv` | Wikipedia photo thumbnail + one-line description + page URL per (race, candidate slot), from the REST summary API with conservative wrong-person guards (`src/data/build_candidate_meta.py`). Consumed by the site's race-page candidate cards; regenerate after roster changes. |
| `ei_env_swings.json` | Ecological-inference per-group vote-margin swings (drives the demographic environment). |
| `fred_cache.json` | Cached FRED economic series (value + fetch timestamp). |
| `leg_historical.csv` | Per-SLD 2016/2020/2024 presidential vote totals and two-party D−R margins; drives `lean.compute_lean` for state-legislative districts. |
| `leg_demographics.csv` | Per-SLD voting-age population race/ethnicity composition (percentage shares). |
| `leg_results.csv` | Normalized candidate-level 2020/2022/2024 state-legislative election results with per-seat derived fields (`leg_margin`, `winner_party`, `contested`). |
| `leg_candidate_strength.csv` | Per-candidate strength score (signed D−R, federal-overperf formula); composite across available cycles. |
| `leg_candidates_2026.csv` | 2026 SLD candidate roster (Ballotpedia scrape + graceful degradation from `leg_results`), with per-slot strengths attached. |
| `leg_members.csv` | Current state-legislative roster (one row per seat) from MultiState: functional party, 2024 presidential margin, next-election year. |
| `leg_races_2026.csv` | 2026 state-legislative contests from MultiState: presidential margin, incumbent + party, open/contested, candidate names, `num_seats`, Cook/Sabato ratings. |
| `leg_chamber_meta.csv` | Authoritative per-chamber totals from MultiState: composition (D/R/Other), `seats_up_2026`, supermajority threshold, and `control` (`D`/`R`/`Coalition`/`Power_Sharing`) with `coalition_desc`. |
| `leg_calibration.json` | Coefficients fit to the 2020/2022/2024 results (`src/model/leg_calibration.py`): `beta_lean`, `beta_inc`, `strength_weight`, `incumbent_hold`, `sigma`, per-cycle environments. |
| `leg_strength_overrides.csv` | Curated per-candidate strength overrides (`candidate,state,strength,note`) applied at full weight, for ticket-splitters with no contested history to auto-measure (e.g. Sam Sutton, NY SD-22). |
| `leg_overrides_sd.csv` | Curated SD general-election roster (SD SOS certified list, post June-2026 primary): one row per SEAT — the 33 two-member at-large House districts split into paired A/B seats, 26A/26B/28A/28B as their own single-member subdistricts. Replaces MultiState's stale pre-primary SD rows via `apply_seat_rosters()`; incumbency is re-derived from 2024 winners in `leg_results.csv`. |
| `leg_overrides_az.csv` | Curated AZ House per-seat roster (30 two-member at-large districts split into A/B seats; PRESUMPTIVE until the Jul 21 primary — Ballotpedia/AZ SOS). Carries explicit `incumbent`/`incumbent_party` columns (covers mid-term appointees). Applied by `apply_seat_rosters()`; the AZ Senate stays on MultiState rows. |

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

### race_polls.csv — per-race NYT matchup polls
One row per poll-question, scraped from the NYT Senate/House/Governor feeds and
keyed to `race_code`. Multi-candidate aware so independents, third parties, and
ranked-choice / top-two contests are represented (ranked-choice polls keep the
final round).
- `race_code`, `poll_id`, `pollster`, `end_date`, `n`, `rating`, `lean`.
- `dem`, `rep` (the dominant Dem/Rep shares), `margin` (`dem − rep`).
- `oth`, `oth_party`, `oth_name` — strongest minor/independent candidate.
- `dem2`, `rep2` — secondary same-party shares (open primaries / top-two).
- `dem_name`, `rep_name` — matched candidate names.

The forecast blends a point-in-time, recency- and quality-weighted average of
these (polls with `end_date ≤` the run date) with each race's fundamentals.

### pollster_ratings.csv — 85 rows
Pollster-quality lookup: `pollster`, `rating` (higher = better), `lean`
(house-effect direction; blank if none), `banned` (`1` = fabrication/fake
pollster excluded from every poll average, rating set to 0 to denote it).

### third_parties.csv — 99 rows
Third-party / independent candidates on the 2026 ballot, detected from FEC
filings: `race_code`, `party`, `name`, `fec_id`.

### ei_env_swings.json
Per-race-group recent vote-margin swing (D-R points; e.g. `hispanic: -23.1`),
estimated by **ecological inference** over the 435 House districts (district
two-party Dem share ~ VAP composition, 2020 vs 2024; bounded MAP fit with a
ridge prior so small/collinear groups don't blow up). Regenerate with
`python -m src.model.ecological`.

This drives the **demographic environment**: a district's composition-weighted,
nationally-centered swing is applied with a **reversion** weight
(`ENV_FEED_WEIGHT = -0.5`) — i.e. groups that swung in 2024 (Hispanics hard R)
are projected to *snap back* in the 2026 midterm. The result feeds each race's
`fundamentals`/`margin` (not just the reported `environment`).

### leg_historical.csv — 6,775 rows (one per 2026 state legislative district)
Presidential vote history for every SLD that appears on the 2026 ballot; produced
by `src/data/load_stateleg.py` from the 2026 district shapefiles' `.dbf` attribute
tables (presidential results pre-attached to each district).
- `code` — canonical district key (e.g. `AK-H-33`; uses `H`/`S` abbreviation, not
  `lower`/`upper`).
- `geoid` — TIGER-line GEOID.
- `state` — two-letter state.
- `chamber` — `lower` / `upper`.
- `district` — district label as a string token (numeric or lettered, e.g. `33`, `A`).
- `pres16_total`/`pres16_dem`/`pres16_rep`, `pres20_*`, `pres24_*` — raw presidential
  votes in the district.
- `margin_2016`/`margin_2020`/`margin_2024` — two-party D−R margin in percentage
  points (negative = Republican advantage). Same convention as `historical.csv`.

### leg_demographics.csv — 6,775 rows (one per SLD)
Race/ethnicity composition of the voting-age population for each SLD; produced by
`src/data/load_stateleg.py` from the same district shapefiles' `.dbf` attributes.
- `code`, `state`, `chamber`, `district` — same keys as `leg_historical.csv`.
- `vap_total` — total voting-age population.
- `vap_white_pct`/`vap_black_pct`/`vap_hisp_pct`/`vap_asian_pct`/`vap_native_pct`/
  `vap_pac_pct`/`vap_minor_pct` — VAP share by race/ethnicity (0–100 percentage
  points; `vap_minor_pct` = all non-white groups combined).

### leg_results.csv — 38,160 rows (one per candidate × election year)
Normalized candidate-level state-legislative results for 2020, 2022, and 2024;
produced by `src/data/prep_leg_results.py`.
- `year` — election year (2020, 2022, or 2024).
- `state`, `chamber`, `district` — geography keys; `district` is a string token.
  `chamber` uses `lower`/`upper` (not the `H`/`S` abbreviation in `code`).
- `seats` — seats contested in the district (usually 1; multimember districts > 1).
- `candidate` — raw candidate name string.
- `party_code` — `d` / `r` / `i` / other.
- `votes` — raw vote count.
- `won` — `True` if the candidate won.
- `contested` — `True` if both a Democrat and a Republican appeared on the ballot.
- `winner_party` — party of the seat winner (`d` / `r` / `i`; blank if uncontested).
- `leg_margin` — two-party D−R margin in percentage points for that seat-year
  (positive = Dem advantage; derived from the D and R vote shares; `NaN` if
  uncontested or no two-party vote available).

### leg_candidate_strength.csv — ~15,400 rows (one per unique candidate)
Per-candidate strength score for all state-legislative candidates with at least
one contested appearance in 2020–2024; produced by `src/data/leg_strength.py`.
- `candidate` — raw candidate name (lower-case, "last, first" as stored in
  `leg_results.csv`).
- `name_key` — normalized "first last" form used for roster matching.
- `state` — two-letter state.
- `party` — `d` / `r`.
- `cycles` — number of election cycles with contested results included in the
  composite (1–3).
- `strength` — composite strength score: signed Democratic-centric (positive =
  outperforms partisan baseline for a Dem; negative = underperforms for a Dem or
  overperforms for a Rep). Formula: `(D% − R%) − pres_lean` per cycle, with ±3-pt
  incumbency discount applied, averaged unweighted across cycles. Scale is
  percentage points.

### leg_candidates_2026.csv — one row per SLD × candidate slot
2026 state-legislative candidate roster covering every district in
`leg_historical.csv`; produced by `src/data/scrape_leg_candidates.py` (Ballotpedia
scrape with graceful degradation to `leg_results` incumbency when scrape fails).
- `code`, `state`, `chamber`, `district` — same keys as `leg_historical.csv`.
- `seats` — number of seats contested.
- `dem_name`/`rep_name`/`other_name` — candidate names in each slot (blank when
  no candidate is known).
- `incumbent_running` — `True` if the incumbent is seeking re-election.
- `incumbent_party` — `d` / `r` / `i` / blank.
- `open_seat` — `True` if no incumbent is running.
- `contested` — `True` if both a Dem and a Rep appear on the ballot.
- `status` — data confidence: `confirmed` (Ballotpedia), `projected` (degraded from
  last `leg_results` winner), or `presumptive`.
- `source` — `ballotpedia` or `leg_results` (degraded fallback).
- `dem_strength`/`rep_strength` — candidate strength scores joined from
  `leg_candidate_strength.csv` by `(state, name_key)`; `0.0` when no history exists.

## Output

Generated forecast output lives in [`output/`](output/). Each run updates the
per-race timelines and the latest-run aggregates, and archives a dated snapshot
under `output/historical/`.

### Per-race timelines

- `output/races/<RACE_CODE>.csv` — **one row per run date**, tracking the race's
  evolution over time: `margin`, `dem_prob`/`rep_prob`, `rating`, `lean`, `gcb`,
  `fundamentals`, `pres_2024`, `swing_24_26`, vote shares, RCV / three-way /
  same-party flags, and independent + third-party fields. Plus:
  - **Per-race polling:** `poll_dem`, `poll_rep`, `poll_count`, `polling_avg`,
    `polling_weight` — the point-in-time race-poll average and how heavily it was
    blended with fundamentals (blank/0 when no race polls exist as of that date).
  - **Localized environment + approval:** `environment` and `demo_env_adj` (the
    EI demographic-reversion adjustment, which now feeds the race `margin`),
    `trump_approve`, `trump_net`.
  - **Demographics:** the Dem two-party share for every demographic group
    (`dem_vap_white`, `dem_age_20_29`, `dem_educ_bachelor`, `dem_inc_200k_plus`, …).

### Latest-run aggregates (overwritten each run)

- `output/races_summary.csv` — one row per race.
- `output/chambers.csv` — per-chamber simulated seat distribution and control
  probabilities.
- `output/national_indicators.csv` — generic ballot (naive vs corrected),
  approval, fundamentals.
- `output/area_indicators.csv` — per House district and state: lean, GCB,
  environment, and localized Trump approval.
- `output/votes.csv` — race-level multi-way vote totals + shares (Dem/Rep/Ind/Other).
- `output/breakdown.csv` — per race × demographic group: Dem/Rep/Ind/Other votes
  + shares (handles three-way / RCV / top-two splits).
- `output/race_polls.csv` — the as-of per-race poll averages (incl. `oth`/`dem2`/`rep2`).
- `output/county_projections.csv` — county-level D/R/IND/OTH vote projection
  (latest run only).
- `output/county_approval.csv` — Trump approval per county (latest run only).
- `output/rcv_rounds.csv` — round-by-round elimination for the six RCV races.

### Timelines & history

- `output/timeline.csv` — daily national + chamber numbers across the cycle.
- `output/historical/<YYYY-MM-DD>/` — per-date archive of that run: `races_summary.csv`,
  `chambers.csv`, `national_indicators.csv`, `area_indicators.csv`, `rcv_rounds.csv`,
  `votes.csv`, `breakdown.csv.gz` (per race × demographic group, gzipped), and
  `race_polls.csv` (the as-of race-poll averages). County-level files are not
  archived per date; per-race history lives in the `races/` timelines.

### State-legislative chamber forecast

A separate stage (`python3 -m src.pipeline.run_stateleg`) forecasts every state
legislative chamber holding a 2026 election. Its **inputs** come from MultiState's
per-state JSON (`src/data/scrape_multistate.py`) plus the calibration fit:

- `leg_members.csv` — current roster, one row per seat (state, chamber, district,
  functional party, 2024 presidential margin, next election year).
- `leg_races_2026.csv` — the 2026 contests: presidential margin, incumbent + party,
  open/contested, `num_seats`, candidate names (`dem_name`/`rep_name`/`incumbent`),
  and Cook/Sabato ratings where available.
- `leg_chamber_meta.csv` — authoritative per-chamber totals (composition,
  `seats_up_2026`, supermajority threshold).
- `leg_calibration.json` — coefficients fit to the 2020/2022/2024 results
  (`src/model/leg_calibration.py`): `beta_lean` (down-ballot carry of presidential
  lean), `beta_inc` (incumbency points), `sigma` (seat residual SD), per-cycle
  environments, `R²`.

Each seat's projected margin mirrors the federal `fundamentals` form —
`beta_lean·(pres − national) + beta_inc·incumbency + env` — where `env` is the
federal generic ballot (`national_indicators.csv:gcb_margin`), localized per state
by the federal Gov/Sen swings (`races_summary.csv:swing_24_26`). **Outputs:**

- `output/leg_chambers.csv` — one row per chamber: current/projected composition,
  `dem_control_prob`, seat distribution (mean/median/90% CI), projected control, flip.
  `cur_control` uses MultiState's authoritative arrangement, so bipartisan-coalition
  chambers (both Alaska) read `Coalition` with a `coalition_desc`, and MN-House reads
  `Power_Sharing`, rather than a partisan seat-count.
- `output/leg_races.csv` — one row per 2026 seat: candidates, the margin breakdown
  (`fed_lean (2016/2020/2024 blend, or pres_margin) → lean + state_swing + inc_adj +
  cand_strength + inc_hold + env = margin`), `dem_prob`/`rep_prob`, and a
  Safe/Likely/Lean/Toss-up `rating`. `cand_strength` is the ticket-splitting term;
  `inc_hold` is a bounded entrenchment bonus for a running incumbent with no measured
  strength (the "virtually uncontested" case). Seats with only one major-party
  candidate are flagged `uncontested` (`D`/`R`) and go to that candidate regardless of lean.
- `output/leg_timeline.csv` — national daily trajectory across the cycle (D/R chambers
  controlled, expected D-controlled, environment) from `--backdate`.
- `output/leg_chamber_timeline.csv` — per chamber per date (`dem_control_prob`,
  projected seats), the historical record behind each chamber.

Produced by the ElectIndex 2026 U.S. forecast pipeline. Inputs are curated from
public sources; the per-race timelines and `historical/` snapshots accumulate
over time, while the top-level aggregates reflect the most recent run.
