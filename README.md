https://whcavender14.github.io/cfb-power-index/#simulations

CFB Power Index — Model Overview
What This Model Does
The CFB Power Index is a predictive college football ratings system that produces:

Weekly power ratings (offense, defense, and combined) for all 138 FBS teams
Season-long simulations (1,000 Monte Carlo trials) producing playoff probabilities, conference championship odds, and win-total distributions
Betting-line analysis comparing model predictions to market spreads and identifying potential value
The system is updated weekly during the college football season via automated pipeline, with all results served as static JSON from GitHub Pages and rendered in a React dashboard (no database or application server required).

Core Methodology
Power Ratings Model (Version 5 with EB Features)
The power ratings blend two complementary signals:

1. Current-Season Form (Data-Driven)
Each week, the model solves a penalized ("ridge") regression on all FBS game scores played so far in the season:

Input: every game score from the current season (score differential = home offense + home defense − away offense − away defense + HFA)
Unknown: the offense and defense strength of each team
Method: find the offensive and defensive ratings that best explain observed game outcomes, with a penalty (ridge shrinkage) that prevents extreme estimates early in the season when evidence is thin
For example, if Team A beats Team B 24–14, that difference of 10 points is modeled as roughly equal to:

A_offense + (−B_defense) − HFA (or + HFA if away)
Solving across all games simultaneously determines each team's offensive and defensive strength in points per game relative to average.

The ridge penalty trades a small amount of bias for significant noise reduction—especially important early season when some teams may have played only 1–2 games. The penalty strength is tuned via nested, forward-in-time validation to prevent overfitting.

2. Preseason Prior (Historical Expectation)
For each team in week 1 of the season, a preseason model predicts where they should start, using training from all historical FBS seasons (2015–2025, excluding 2020):

Predictors: prior year's offense/defense rating, returning player production, recruiting talent composite, blue-chip ratio, coaching tenure/new-coach status, transfer portal net activity
Validation: strictly forward-in-time — predictions for year N are trained only on seasons 1 through N−1, never on season N itself
Output: separate offensive and defensive priors, which anchor the blend early in the season
3. The Blend (Adaptive Handoff)
The model combines current form and preseason prior in a ratio that adapts to the calendar:

Early season (few games played): preseason prior dominates
Late season (many games played): current form dominates
Tuning: the handoff rate is a parameter that is itself fit via cross-validation
By week 14, most teams have played ~11 games, and the blend is heavily weighted toward observed performance.

The final power rating is:

Power Rating = Offense Rating − Defense Rating
(Lower defensive ratings are better; power represents "how many points better than average this team is.")

Home-Field Advantage
Home-field advantage is estimated from the data, not assumed:

The model includes an HFA offset in the regression as an unknown to be learned
Typical 2026 season value: ~3.07 points
Estimated separately for each season, since HFA can vary year-to-year
Treated as constant across all teams (no team-specific home advantage)
Betting Line Integration
The third dashboard tab (Betting Line Analysis) displays:

Model prediction: away power − home power − HFA (negative favors home)
Market spread: fetched from sportsbook APIs (DraftKings preferred, with ESPN Bet, Bovada, Caesars, and consensus as fallbacks)
Signed discrepancy: model line minus market line (negative suggests market favors home side more than the model)
This is not a probability prediction or a recommendation to bet—it is purely a comparison of how the model's ratings differ from the closing sportsbook consensus. The model never uses Vegas lines as an input; betting lines are used only as an external validation benchmark.

Data & Validation
Training Data
Historical seasons: 2015, 2016, 2017, 2018, 2019, 2021, 2022, 2023, 2024, 2025 (COVID 2020 excluded due to irregular schedules)
Current season: 2026, refreshed weekly through the bowl games
Coverage: all FBS teams and FCS opponents in regular-season games; FCS-only games are excluded
Leakage Auditing
Strict time-travel validation ensures no information from the future is used to predict the past:

Parameters tuned from historical seasons are frozen before applying to 2026
Weekly ratings use only game results available before each game kicked off (not the same-week results)
Preseason models are trained on prior years only
Confidence & Calibration
Margin predictions are validated against 4 recent seasons (2023–2025, conditional test set)
Bootstrapped confidence intervals resample entire seasons (acknowledging that games within a season are correlated)
Calibration checked: are 65%-win-probability teams actually winning 65% of simulated instances?
Residual distribution: calibrated to approximately 15.79 points SD around predictions (tuned from training data)
Season Simulation
Approach
The simulation engine uses cfbseedR with the production power ratings as the results generator:

Monte Carlo trials: 1,000 independent simulations of the full 2026 season
Game-by-game results: for each simulated game:
Home team expected score = home_power − away_power + HFA + random residual
Away team expected score = away_power − home_power − HFA + random residual
Residuals sampled from calibrated normal distribution (SD ≈ 15.79 points)
Winner determined by which margin exceeds zero
Playoff bracket: automatic CFP bracket seeding using actual CFB selection criteria:
Top 4 seeds: P5 conference champions (highest-rated teams)
Remaining slots: best non-champions and independent teams
Conference championship games required for P5 champions (modeled as week 15–16)
Outputs
For each team across all 1,000 trials:

Win-loss record distribution: probability of 8–9–10 wins, etc.
Playoff probability: P(makes top 4)
National championship probability: P(wins bowl/playoff chain)
Conference champion probability: P(wins conference title)
FCS opponents are modeled as a fixed -25 power rating (league-average FCS team).

Model Refinement & Testing
The model undergoes structured validation before promotion to production:

Production Candidate Selection (Gate Process)
A new model variant must clear several gates to be promoted from experimental to frozen:

Integrity & Coverage: 50+ code and logic checks; 100% team/game coverage
MAE Improvement: ≥0.25 points on both development and conditional test sets
Bootstrap Bound: 95% confidence interval for improvement excludes zero
Calibration Metrics: slope in [0.90, 1.10], SD ratio in [0.85, 1.15]
Bias Checks: P4-vs-G5 games, special opponent classes not significantly worse
Ablation Testing: individual components validated independently
Current Frozen Model
Candidate: EB_features (Empirical Bayes with contextual features)
Promotion history: selected in Round 5; tested against multiple enhancement attempts in Rounds 7–10
Conditional MAE (2023–2025): 12.52 points (vs. market closing line: 12.00 points)
Development MAE (2018–2019, 2021–2022): 12.97 points
Feature Set (Contextual Inputs)
In addition to game scores and preseason priors, the model optionally incorporates:

Returning production: fraction of offensive/defensive snaps from prior season retained
Recruiting talent: on-cycle recruiting class rating (composite score and blue-chip ratio)
Portal activity: net gain/loss of players via transfer portal
Coaching tenure: seasons at current school; flag for first-year coach
These features are optional—the model safely falls back to score-only mode if a dated snapshot file is unavailable.

Output & Deployment
Data Format
All outputs are versioned as JSON (schema version 1) with metadata:

{
  "season": 2026,
  "week": 5,
  "updated_at": "2026-10-13T09:00:00Z",
  "status": "success",
  "model": "EB_features",
  "design_hash": "...",
  "feature_hash": "...",
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
      "win_probability_distribution": { "8": 0.05, "9": 0.15, ... },
      ...
    }
  ]
}
GitHub Pages Deployment
Weekly refresh: Mondays 09:00 UTC (August–January)
Manual dispatch: available year-round
Workflow: fetch schedule → build ratings → run simulation → export JSON → build React app → deploy
Caching: production snapshots preserved even if ratings or simulation fail
CI/CD: GitHub Actions with R 4.4.3, Node.js 22, cfbseedR pinned to specific revision
Live Dashboard
React + TypeScript + Tailwind CSS frontend
Three tabs: Power Ratings (searchable, ranked), Season Simulations (probabilities), Betting Analysis
Responsive: mobile-friendly table layouts with horizontal scroll
Accessible: keyboard navigation, screen-reader labels, tooltip definitions
Persistent state: search and conference filter choices saved between views
Key Assumptions & Limitations
Ridge regression trades bias for variance: early-season ratings are smoothed toward zero and preseason priors; this reduces noise but means early predictions are slightly regressed.

Constant HFA across teams: the model assumes home field advantage is ~3 points for all teams. Some teams do have genuine home-field effects, but estimating team-specific HFA requires more data than available.

No special team effects: field goal position, turnover rates, penalty flags, and other granular stats are not explicitly modeled; they are implicitly captured in the overall offensive/defensive rating to the extent they affect point differentials.

FCS treated uniformly: FCS opponents are assigned a single fixed power rating rather than being individually estimated.

Stationarity: the model assumes the relationship between scores and underlying talent doesn't change materially within a season (home field advantage is roughly constant, the effect of each additional game is predictable, etc.).

Confidence intervals are descriptive: with only ~10 seasons of training data, bootstrap intervals acknowledge correlation within seasons but should not be interpreted as precise statistical guarantees.

Performance Benchmarks
Metric	Value	Notes
Conditional MAE (2023–2025)	12.52 points	2,398 games; clean test set
Development MAE (2018–2019, 2021–2022)	12.97 points	3,092 games; used for parameter tuning
Closing market MAE	12.00 points	sportsbook consensus benchmark
Calibration slope	0.97	near-perfect, slight underconfidence
SD ratio	0.80	model variance 20% lower than observed; implies small overconfidence
Residual SD	15.79 points	typical prediction error margin
Win probability calibration	~0.95	95%-probability teams win ~95% of simulated games
Running Locally
Prerequisites
R 4.4.3+ with packages: dplyr, tidyr, purrr, tibble, rlang, ggplot2, jsonlite, Matrix, quantreg, cfbfastR, remotes
cfbseedR (pinned to 4a1c78e… via remotes)
Node.js 22 with pnpm 11.19.0
CollegeFootballData API key (in CFBD_API_KEY env var)
Commands
# Export current ratings and simulations (no API calls, uses frozen snapshots)
CFB_SEASON=2026 Rscript scripts/export_public_data.R

# Full refresh: fetch live schedule, update ratings, simulate season
CFB_SEASON=2026 CFB_REFRESH_SCHEDULE=true Rscript scripts/weekly_refresh.R

# Update betting lines only (uses current production ratings)
CFB_SEASON=2026 Rscript scripts/export_betting_data.R

# Build and serve React dashboard locally
pnpm install --frozen-lockfile && pnpm dev
References
Power rating methodology: ridge regression with adaptive preseason blend; see comments in cfb_power_ratings_vCurrent.R
Simulation engine: cfbseedR (https://github.com/sportsdataverse/cfbseedR)
Data source: CollegeFootballData (https://collegefootballdata.com)
Validation audits: see archived round reports (e.g., archive/v10-round10/docs/REPORT.md) for detailed gate results and ablation tests
