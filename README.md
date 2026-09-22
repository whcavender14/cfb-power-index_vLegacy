# CFB Power Index — Model Overview

https://whcavender14.github.io/cfb-power-index/

## 🎯 What This Model Does

The **CFB Power Index** is a predictive college football ratings system that produces three core outputs:

| Output | Description |
|--------|-------------|
| **Weekly Power Ratings** | Offensive, defensive, and combined ratings for all 138 FBS teams, updated every Monday |
| **Season Simulations** | 1,000 Monte Carlo trials producing playoff probabilities, conference championship odds, and win-total distributions |
| **Betting Analysis** | Model predictions vs. market spreads (DraftKings, ESPN, Bovada consensus) |

The system runs **fully automated via GitHub Actions** every Monday at 09:00 UTC (August–January), fetches the current schedule, rebuilds ratings, runs simulations, and publishes static JSON to GitHub Pages. The React dashboard requires **no database or application server**—all data is versioned in Git and served as static files.

---

## 📊 Core Methodology

### Power Ratings Model (Version 5 with EB Features)

The power ratings **blend two complementary signals**, each updated weekly:

#### 1️⃣ **Current-Season Form** (Data-Driven)

Each week, the model solves a **penalized ("ridge") regression** on all FBS game scores played to date:

**Inputs:**
- Every FBS game score from the current season
- Score differential modeled as: `home_off + home_def − away_off − away_def ± HFA`

**Unknowns:** Each team's offensive and defensive strength (in points/game relative to average)

**Method:** Simultaneously solve for all 138 teams' ratings that best explain observed outcomes, with a **ridge penalty (shrinkage)** that:
- Prevents extreme early-season estimates when evidence is thin
- Trades a small bias for significant noise reduction
- Penalty strength is tuned via nested, forward-in-time validation

**Example:**
```
Team A defeats Team B 24–14 (margin = +10)
Is modeled as: A_offense + (−B_defense) − HFA ≈ +10
```

Solving across all games simultaneously yields each team's true strength.

**Why ridge regression?** Early in the season, a team might have only 1–2 games. A raw estimate from 1–2 games is extremely noisy. Ridge shrinkage pulls those extreme estimates toward zero and the preseason prior until enough evidence accumulates. This trades a small bias for a large noise reduction.

#### 2️⃣ **Preseason Prior** (Historical Expectation)

For week 1 of the season, a **preseason regression model** predicts each team's starting ratings using historical FBS data (2015–2025, excluding 2020):

**Predictors:**
- Prior year's offensive/defensive rating
- Returning player production (% snaps retained)
- Recruiting talent composite and blue-chip ratio
- Coaching tenure and first-year coach flag
- Transfer portal net activity (in vs. out)

**Validation:** Strictly forward-in-time — predictions for season N are trained *only* on seasons 1 through N−1, never using season N outcomes

**Output:** Separate offensive and defensive priors that anchor the blend at week 1

#### 3️⃣ **The Adaptive Blend** (Handoff from Priors to Data)

The model combines current form and preseason prior in a **dynamic ratio that adapts to the calendar**:

| Season Phase | Blend Ratio | Why |
|---|---|---|
| **Week 1–2** (0–2 games) | 90% prior, 10% data | Preseason dominates; little game evidence |
| **Week 4–6** (3–5 games) | 60% prior, 40% data | Data gaining influence |
| **Week 10+** (9+ games) | 20% prior, 80% data | Current form dominates |

The **handoff rate itself is a tuned parameter** fitted via cross-validation to optimize predictions.

**Final Rating Calculation:**
```
Power Rating = Offensive Rating − Defensive Rating
```

Where:
- **Power Rating** = points better/worse than average
- **Negative defensive rating** = strong defense (removes points from opponent score)
- Example: Offense +25, Defense +5 → Power +20 (20 points better than average)

---

### 🏟️ Home-Field Advantage (Data-Driven)

Home-field advantage is **estimated from the data**, not assumed:

- Included in the regression as an unknown to be learned
- **2026 estimate:** ~3.07 points
- Estimated *separately for each season* (HFA can vary year-to-year)
- Treated as **constant across all teams** (no team-specific HFA; the model doesn't estimate that Iowa has a +4 edge while Georgia has +3)

**Why include it?** Accounting for HFA prevents it from inflating home teams' ratings and deflating away teams' ratings.

---

### 📈 Betting Line Integration

The dashboard's **Betting Line Analysis** tab displays:

**Model Prediction:**
```
Model Line = Away Power − Home Power − HFA
```
(Negative = home team favored)

**Market Data:**
- Fetched from sportsbook APIs (DraftKings preferred → ESPN Bet → Bovada → Caesars → consensus)
- Updated Monday after weekly refresh
- Displayed provider and retrieval timestamp

**Signed Discrepancy:**
```
Discrepancy = Model Line − Market Line
(Negative = market favors home more than the model)
```

**Important:** This tab is **validation only**, not a betting recommendation:
- ✅ Model lines never trained on Vegas spreads (no circular reasoning)
- ✅ Betting lines used only as external benchmark to check if model differs from market
- ✅ Comparison shows where model estimates diverge from collective market wisdom
- ❌ Not meant as probability prediction or wagering advice

---

## 📋 Data & Validation

### Training Data

| Dataset | Years | Notes |
|---------|-------|-------|
| **Development** | 2018, 2019, 2021, 2022 | Used for tuning model parameters (~3,092 games) |
| **Conditional (Test)** | 2023, 2024, 2025 | Clean test set for validation (~2,398 games) |
| **Historical** | 2015–2019, 2021–2025 | Full history; 2020 excluded (COVID irregular schedules) |
| **Current** | 2026 (weekly refresh) | Updated every Monday through bowl games |
| **Coverage** | All 138 FBS teams | Includes FCS opponents in cross-conference games |

### ⚠️ Leakage Auditing (Time-Travel Validation)

**Strict forward-in-time rules prevent using future information to predict the past:**

- ✅ Parameters tuned from 2018–2022 are **frozen** before applying to 2026
- ✅ Weekly ratings use only results **before each game kicked off** (not same-week results)
- ✅ Preseason models trained on *prior years only* (predictions for 2026 never train on 2026 data)
- ✅ Candidate selection gates use only development/conditional splits (never "what works on all data combined")

**Why this matters:** Without time-travel discipline, you can fool yourself into thinking your model is great when it's really just memorizing history.

### 🎯 Confidence & Calibration

**Validation Approach:**
- Margin predictions tested on **2023–2025** (recent, realistic conditions)
- Bootstrapped confidence intervals **resample entire seasons** (respects correlation within seasons)
- Calibration check: **65%-win-probability teams actually win ~65%** of simulated games?
- Residual SD: ~**15.79 points** (typical prediction error; tuned from training data)

**Confidence Intervals:** Described as "descriptive" rather than statistical guarantees (only ~10 seasons of history makes precise inference difficult)

---

## 🎲 Season Simulation

### Approach

The simulation engine uses **cfbseedR** with production power ratings as the results generator:

**Step 1: Monte Carlo Trials**
- **1,000 independent simulations** of the full 2026 season
- Each trial plays out all remaining games from the current cutoff

**Step 2: Game-by-Game Results**

For each simulated game:
```
Home expected score = home_power − away_power + HFA + residual
Away expected score = away_power − home_power − HFA + residual

Winner = team with higher expected score
```

Where:
- `home_power / away_power` = team's offensive rating minus defensive rating
- `HFA` = home-field advantage (~3.07 points)
- `residual` = random noise from calibrated normal distribution (SD ≈ 15.79 points)

**Step 3: Playoff Bracket**

Automatic CFP seeding using **actual FBS criteria:**
- **Top 4 seeds:** P5 conference champions (ranked by power rating)
- **Remaining slots:** best non-champions and independent teams
- **Conference championships:** modeled as week 15–16 for P5 teams

### 📊 Simulation Outputs

For each of the 138 FBS teams:

| Metric | Example | Interpretation |
|--------|---------|---|
| **Win probability distribution** | P(8 wins) = 12%, P(9) = 25%, P(10) = 30%, ... | What are the odds they finish with 8, 9, 10 wins? |
| **Playoff probability** | 85% | Chance of making top-4 playoff |
| **Championship probability** | 4% | Chance of winning national title (including bowl wins) |
| **Conference title probability** | 12% | Chance of winning conference championship |

**FCS Opponents:** Modeled with fixed **-25 power rating** (representative FCS average strength)

---

## 🔬 Model Refinement & Testing

All new model variants undergo **structured gating** before promotion to production:

### Promotion Gate Process

A candidate must clear **all gates** to move from experimental → frozen production:

| Gate | Requirement | Pass/Fail |
|------|-------------|-----------|
| **Integrity** | 50+ code/logic checks; 100% coverage | ✅ |
| **MAE Improvement** | ≥0.25 points on *both* development and conditional | ❌ if either fails |
| **Bootstrap CI** | 95% confidence interval excludes zero | ❌ if CI contains zero |
| **Calibration Slope** | Must be in [0.90, 1.10] | ❌ if outside range |
| **Calibration SD Ratio** | Must be in [0.85, 1.15] | ❌ if outside range |
| **P4-vs-G5 Bias** | Can't be worse than incumbent | ❌ if worse |
| **Ablation Tests** | Individual features don't hurt performance | ❌ if feature hurts |

**Example:** Round 10 candidate improved development MAE by 0.10 but *failed* conditional MAE (−0.001, 95% CI [−0.038, +0.022]). Failed on gates 2 and 3 → not promoted.

### 📌 Current Production Model

| Attribute | Value |
|-----------|-------|
| **Name** | `EB_features` (Empirical Bayes + contextual) |
| **Status** | Frozen production; selected Round 5 |
| **Conditional MAE (2023–2025)** | **12.52 points** |
| **Development MAE (2018–2022)** | **12.97 points** |
| **vs. Market (closing line)** | Market: 12.00 points; model: +0.52 pts worse |
| **Seasons tested** | Rounds 7–10: attempted improvements, none passed gates |

---

## 🎁 Feature Set (Optional Contextual Inputs)

Beyond game scores and preseason priors, the model *optionally* incorporates contextual features:

| Feature | Source | Purpose |
|---------|--------|---------|
| **Returning Production** | Team sports data | % of offensive/defensive snaps retained from prior season |
| **Recruiting Talent** | On-cycle recruiting composite | Recruiting class rating (composite score + blue-chip %) |
| **Portal Activity** | Transfer portal database | Net player inflow/outflow for the season |
| **Coaching Tenure** | Public records | Years at current school; flag for first-time head coach |

**Graceful Fallback:** If a dated feature file is unavailable, the model safely reverts to **score-only mode** rather than guessing. This prevents data staleness from corrupting predictions.

---

## 📤 Output & Deployment

### JSON Data Format (Schema v1)

All outputs versioned as JSON with metadata envelope:

```json
{
  "season": 2026,
  "week": 5,
  "updated_at": "2026-10-13T09:00:00Z",
  "status": "success",
  "model": "EB_features",
  "design_hash": "abc123def...",
  "feature_hash": "xyz789...",
  "hfa_points": 3.07,
  "teams": [
    {
      "team_id": 25,
      "team": "Ohio State",
      "conference": "Big Ten",
      "power_rating": 18.5,
      "off_rating": 24.2,
      "def_rating": 5.7,
      "games_played": 5,
      "playoff_probability": 0.92,
      "conference_champion_probability": 0.45,
      "national_champion_probability": 0.08,
      "win_distribution": {
        "8": 0.02,
        "9": 0.08,
        "10": 0.18,
        "11": 0.35,
        "12": 0.28,
        "13": 0.07
      }
    },
    ...
  ]
}
```

**Stored Locations:**
- `public/data/ratings.json` — latest ratings snapshot
- `public/data/simulations.json` — latest simulation snapshot
- `public/data/<season>/week-<NN>/` — historical week archives (full history in Git)

### 🚀 GitHub Actions Automation

| When | What | Notes |
|------|------|-------|
| **Mondays 09:00 UTC** | Auto-refresh | Aug–Jan only |
| **Manual dispatch** | On-demand run | Year-round, via Actions tab |
| **Each run:** | 1. Fetch current schedule<br>2. Build power ratings<br>3. Run 1,000 simulations<br>4. Export JSON<br>5. Build React app<br>6. Deploy to Pages | ~15 min total |

**Resilience:**
- ✅ Ratings fail → site keeps old data, explicit failure message
- ✅ Simulations fail → site publishes valid ratings with "unavailable" simulation state
- ✅ Missing individual metric → field is `null`, not fabricated

**Stack:**
- R 4.4.3 + cfbseedR (pinned to SHA `4a1c78e…`)
- Node.js 22, pnpm 11.19.0
- GitHub Pages (no database, no app server)

### 💻 Live Dashboard

**Frontend Stack:** React + TypeScript + Tailwind CSS

**Three Tabs:**

| Tab | Content | Features |
|-----|---------|----------|
| **Power Ratings** | Weekly team ratings (OFF, DEF, POWER) | Searchable, sortable, conference-filterable, ranked 1–138 |
| **Season Simulations** | Playoff/championship probabilities | Win-distribution charts, heat maps by seed/outcome |
| **Betting Analysis** | Model vs. market spreads | Provider, quote age, line comparison, discrepancy |

**Accessibility & UX:**
- 📱 Mobile-responsive (horizontal scroll in tables)
- ♿ Screen-reader labels, keyboard navigation, skip links
- 🎨 Persistent state (search/filter choices saved between views)
- 🖼️ Graceful logo fallback (shows team initials if image fails)
- 📊 Numeric sorting with nulls always last

---

## ⚠️ Key Assumptions & Limitations

### Methodological Choices

| Assumption | Rationale | Tradeoff |
|-----------|-----------|----------|
| **Ridge regression** | Reduces noise early season when evidence is thin | Early-season ratings are regressed toward priors (less sensitive to small samples) |
| **Constant HFA** | Simple, stable; team-specific HFA requires more data | All teams treated as having ~3-pt home edge; misses genuine team-specific effects |
| **No granular stats** | Keep model simple; avoid multicollinearity from overlapping metrics | Field goal position, turnover rate, penalties implicitly captured only in point diff |
| **Single FCS rating** | Limited data; FCS schedule small relative to FBS | All FCS opponents treated as ~-25 power (average FCS strength); misses individual variation |
| **Stationarity** | Reduce parameter count; easier to validate | Assumes talent/HFA/game-effect don't shift within season |

### Data Limitations

1. **~10 seasons of history** → confidence intervals are "descriptive" not statistically precise
2. **COVID 2020 excluded** → no pandemic-disrupted seasons in training
3. **FCS sample size** → FCS teams individually unrated (only -25 as group)
4. **Portal data incomplete** → feature file requires recent snapshot; falls back to score-only if missing

### What the Model Doesn't Capture

- ❌ **Injuries to star players** — reflected in offensive/defensive rating only after games played
- ❌ **Weather conditions** — neutral site modeling but not weather-specific (cold, rain, wind)
- ❌ **Motivation shifts** — bowl games, rivalry games treated like any other
- ❌ **Play-calling adjustments** — implicit only in opponent's rating, not reactive
- ❌ **Quarterback-specific effects** — lumped into team's offensive rating

### Why These Limitations Matter

These are **accepted tradeoffs**, not bugs:
- Simple models are more auditable and less likely to overfit
- Reproducibility requires frozen training data (can't retrain live on all available info)
- Confidence intervals with ~10 seasons of data are inherently noisy
- "Good enough" predictions are more useful than over-engineered models that fail on edge cases

---

## 📈 Performance Benchmarks (2023–2025 Conditional Test Set)

| Metric | Value | Interpretation |
|--------|-------|---|
| **Mean Absolute Error (MAE)** | **12.52 points** | Typical prediction miss: within ±12.5 points |
| **Development MAE** | 12.97 points | Tuning set performance (2018–2019, 2021–2022) |
| **Market MAE** | 12.00 points | Sportsbook consensus; model 0.52 pts worse |
| **Calibration Slope** | **0.97** | Near-perfect (should be 1.0); slight underconfidence |
| **SD Ratio** | **0.80** | Model variance ~20% lower than observed |
| **Residual SD** | **15.79 points** | Noise around predictions (distribution SD) |
| **Win Prob Calibration** | **~0.95** | 95%-prob teams win ~95% in simulations ✅ |

**Interpretation:**
- ✅ Model is well-calibrated (predicted probabilities match realized frequencies)
- ✅ Predictions realistic (~15.8 point noise) not overconfident
- ⚠️ Model misses market's consensus by 0.5 points on average (market slightly more accurate 2023–25)
- ℹ️ No team-specific bias in top-tier or G5 vs. P4 games

---

## 💻 Running Locally

### System Requirements

| Component | Version | Notes |
|-----------|---------|-------|
| **R** | 4.4.3+ | Core model & simulation |
| **Node.js** | 22 | React build & dev server |
| **pnpm** | 11.19.0 | Package manager (pinned) |
| **cfbseedR** | SHA `4a1c78e…` | Installed via remotes |
| **API Key** | CollegeFootballData | Set as `CFBD_API_KEY` env var |

### R Package Dependencies

```r
install.packages(c(
  "dplyr", "tidyr", "purrr", "tibble", "rlang", "ggplot2",
  "jsonlite", "Matrix", "quantreg", "cfbfastR", "remotes"
))

remotes::install_github(
  "sportsdataverse/cfbseedR@4a1c78e184773c22da43c40a9eb999d23283297f"
)
```

### Common Commands

```sh
# Export ratings & simulations (no API calls; uses frozen snapshots)
CFB_SEASON=2026 Rscript scripts/export_public_data.R

# Full refresh: fetch live schedule, rebuild ratings, resimulate
CFB_SEASON=2026 CFB_REFRESH_SCHEDULE=true Rscript scripts/weekly_refresh.R

# Update betting lines (uses current production ratings)
CFB_SEASON=2026 Rscript scripts/export_betting_data.R

# Validate exported data against schema
Rscript tests/test_public_export.R

# Build & serve React dashboard locally
pnpm install --frozen-lockfile
pnpm dev
```

**Output Locations:**
- `cfb_data/` — cached schedules and live data
- `outputs/round4/` — current rankings CSV
- `public/data/` — exported JSON for dashboard

---

## 📚 References

**Code & Methodology**
- [`cfb_power_ratings_vCurrent.R`](cfb_power_ratings_vCurrent.R) — Annotated power ratings model with detailed comments on ridge regression and validation
- [`cfb_simulation.R`](cfb_simulation.R) — Monte Carlo engine using cfbseedR with production ratings
- [`run_2026_rankings.R`](run_2026_rankings.R) — Weekly entrypoint that fetches schedule, builds ratings, exports JSON

**External Tools**
- **cfbseedR:** Monte Carlo CFP simulations ([sportsdataverse/cfbseedR](https://github.com/sportsdataverse/cfbseedR))
- **Data:** CollegeFootballData.com API ([collegefootballdata.com](https://collegefootballdata.com))
- **React Dashboard:** TypeScript + Vite + Tailwind CSS

**Validation & Testing**
- Gate results & ablations: [`archive/v10-round10/docs/REPORT.md`](archive/v10-round10/docs/REPORT.md)
- Data contract: [`docs/DATA_CONTRACT.md`](docs/DATA_CONTRACT.md)
- Unit/integration tests: [`tests/`](tests/)

**Related Documentation**
- See [`archive/README.md`](archive/README.md) for historical model versions and round progression
- GitHub Actions workflow: [`.github/workflows/site.yml`](.github/workflows/site.yml)
