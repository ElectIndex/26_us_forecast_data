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

## Output

Generated forecast output lives in [`output/`](output/):

- `output/races/<RACE_CODE>.csv` — one CSV per race: topline, demographic
  breakdown, county section, and (for RCV races) round-by-round elimination.
- `output/races_summary.csv` — one row per race.
- `output/chambers.csv` — per-chamber simulated seat distribution and control
  probabilities.
- `output/national_indicators.csv` — generic ballot (naive vs corrected),
  approval, fundamentals.
- `output/area_indicators.csv` — per House district and state: lean, GCB,
  environment, and localized Trump approval.
- `output/county_projections.csv` — county-level D/R/IND/OTH vote projection.
- `output/county_approval.csv` — Trump approval per county.
- `output/rcv_rounds.csv` — round-by-round elimination for the six RCV races.
- `output/timeline.csv` — daily backtest of national + chamber numbers.

Produced by the ElectIndex 2026 U.S. forecast pipeline. Inputs are curated from
public sources; output is regenerated each run.
