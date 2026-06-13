# Learning Log — FinBe

## Session 1 — Setup + first look at the data

Built the whole environment from nothing: installed Anaconda, connected
the Mac to GitHub with a personal access token, cloned the repo, got a
football-data.co.uk CSV into it, and wrote my first notebook.

What tripped me up:
- macOS blocked moving the CSV from Downloads via Terminal ("operation
  not permitted") — had to grant permission / drag it in via Finder.
- Launched Jupyter from the wrong folder, so it couldn't see my files.
  Lesson: jupyter lab opens wherever you run it from — cd into the
  project first.
- GitHub doesn't accept your password for git anymore — needs a token.

What I actually learned:
- Computed the home/draw/away split: 43% / 24% / 33% (League One).
- This is a "baseline" — the dumbest predictor, blind to which teams
  play. Any real model has to beat 43%. You don't get credit for being
  right, you get credit for beating the baseline. (Same logic as beating
  a market price.)

## Session 2 — Team strength numbers

Built a model that actually knows who's playing: for each team,
average goals scored and conceded, split home and away.

What tripped me up:
- The FTHG/FTAG flip in the away calculation. When a team plays away,
  the goals THEY concede come from the HOME goals column (FTHG), not the
  away one. Easy to get backwards, and if you do, everything downstream
  is silently wrong.

What I actually learned:
- Sanity-checked the numbers against the final table myself: Birmingham
  (1st) scores most / concedes least, Charlton (4th) right behind. The
  model "knows" something real about quality, purely from goal data.
- Big insight: an average tells you EXPECTED goals but not the SPREAD.
  "1.3 goals scored" could mean consistently tight games or wild swings
  — same average, totally different match shapes. The mean can't tell
  them apart. That gap is exactly what Poisson is supposed to fix next
  session.

## Session 3 — Poisson model: from strengths to match probabilities

Built the model that turns team strengths into actual match
predictions. The chain: each team's attack/defence strength →
expected goals for a specific matchup → Poisson spreads that into
scoreline probabilities (P of scoring exactly 0,1,2,3...) → combine
home and away distributions into home-win / draw / away-win.

Tested on Birmingham (1st) vs Burton (~20th):
- Expected goals: 2.03 vs 0.33
- Burton's chance of scoring exactly 0: 72%
- Result: Home 77.9% / Draw 17.0% / Away 5.1% (sums to 1.0 — checks out)

What tripped me up / what I learned:
- I predicted 46% home win, model said 78%. The gap was the lesson:
  I anchored on "strong vs average," but this was "best vs worst" —
  the two extremes multiply. My football intuition reasons about one
  team's quality; the model reasons about the GAP between two teams.
- Poisson is what finally closed the average-vs-spread problem I
  spotted in session 2: "expect 2.03 goals" became a full distribution
  over actual scorelines.

The big realisation for next time — IN-SAMPLE vs OUT-OF-SAMPLE:
- Right now I build strengths from a season AND predict matches from
  that SAME season. The model has already seen the answers. That's
  not prediction, it's describing data I already have. A model always
  looks good on the data it was built from.
- Real prediction = build strengths from one period, test on a LATER
  period the model never saw. Train on early, test on later, never let
  it peek. That's the backtest, and it's the only honest way to answer
  "do I have an edge."
- Also noted: team strength isn't fixed (form, transfers, managers
  change). A season-long average is a crude proxy. Real models weight
  recent games more. Refinement for later.

Next session: the train/test split — the start of honest backtesting.

## Session 4 — First honest backtest (train/test split)

The big gear-change: stopped letting the model grade its own homework.

Set-up:
- Parsed dates, sorted the season chronologically.
- Split 70/30 BY DATE: trained on Aug 2024–Feb 2025 (386 matches),
  tested on Feb–May 2025 (166 matches the model never saw).
- Built strengths from TRAIN ONLY, then predicted the test matches.
- Note: League One has 24 teams = 552 matches (46 per team), not 380.
  Worth remembering — it's why every team still had enough games after
  the split (no missing strengths).

What happened to Birmingham vs Burton:
- Full-season model (session 3): 0.779 / 0.170 / 0.051
- Train-only model: 0.712 / 0.220 / 0.068
- The home-win prob DROPPED ~7 points. Why: seeing only games up to
  February = fewer games = less extreme, less confident estimates. The
  model softening is it being honest about knowing less. Good.

The result:
- Model accuracy: 51.2% (85/166)
- Baseline (always pick Home): 40.4%
- So the model BEAT the naive baseline out-of-sample by ~11 points.
  The team-strength signal is real and survives on unseen data. This
  was the thing I worried wouldn't hold up — it did.

The big flaw (more instructive than the win):
- Model predicted 104 H, 61 A, but only 1 DRAW — out of 166.
- Yet 39 of those matches (24%) WERE draws. The model is structurally
  blind to a quarter of all outcomes.
- KEY DISTINCTION: this is a DECISION-RULE flaw, not a model flaw. The
  Poisson model does assign draws real probability (Birmingham-Burton
  got 22%). It's "pick the single highest-probability outcome" that
  discards them, because a draw is almost never the single likeliest
  result even when it's likely in absolute terms.

The deeper lesson — ACCURACY IS THE WRONG METRIC:
- A bookmaker doesn't care if I "called" the result. They care whether
  my probabilities are CALIBRATED: when I say 22% draw, do draws
  actually happen ~22% of the time?
- A model can have mediocre accuracy but great calibration and still be
  profitable. Great accuracy + bad calibration = useless for betting.
- So 51% is just a sanity check that the model isn't broken. It is NOT
  the number that tells me whether I have an edge.

Next session: calibration — the metric that actually matters for betting.

## Session 5 — Calibration (the metric that actually matters for betting)

Moved past accuracy to the real question: when the model says 30%,
does it happen ~30% of the time?

Method:
- Took the 166 out-of-sample home-win predictions from session 4.
- Bucketed them by predicted probability (0-0.2, 0.2-0.3, ... 0.7-1.0).
- In each bucket, compared AVG PREDICTED probability against the
  ACTUAL home-win rate. Close together = well-calibrated.
- Plotted it as a calibration curve (predicted vs actual, with the
  diagonal y=x as "perfect"). Point size = number of matches.

What the curve showed:
- Where it matters most (0.4-0.7, holding the most matches: n=37,30,19)
  the model is WELL-CALIBRATED. Predicted 0.45 / actual 0.41, predicted
  0.54 / actual 0.53, predicted 0.64 / actual 0.63. When it says "55%
  home win," home teams win ~53%. The probabilities are trustworthy
  where most predictions live. Real, non-obvious, encouraging.

Reading the wobble — signal vs noise (the actual skill):
- Extreme buckets looked off but are TOO THIN to judge. The 0.7-1.0
  bucket = 6 matches = "0.667 actual" is just 4 wins out of 6. Flip one
  match and it's 0.83. Meaningless. Cross it out.
- The 0-0.2 bucket (n=15) also too thin to trust.
- The ONE real finding: 0.2-0.3 bucket has 35 matches (a real sample)
  and predicted 0.25 vs actual 0.11. Genuinely OVERCONFIDENT when it
  thinks the home team is a moderate underdog — it's not pessimistic
  enough about them. Worth investigating, not panicking.

The meta-lesson (biggest takeaway):
- Half my buckets are too small to conclude anything. 166 matches isn't
  enough. The single biggest improvement to this whole project is NOT a
  cleverer model — it's MORE DATA. Multiple seasons, multiple leagues.
  That fills every bucket and turns "hint of overconfidence" into
  "confirmed and measurable."
- Filed: next major leap = data volume, not model complexity.

Why this connects to the real goal:
- A bookmaker's odds imply a probability. To find value I need my
  probability to be both DIFFERENT from theirs AND more accurate.
  Calibration is what tells me whether my probabilities can be trusted
  to do that. It's the bridge to comparing against the market.

Next session: either (a) load multiple seasons to fix the data problem,
or (b) start the odds-side work — overround and margin removal (Buchdahl).

## Session 6 — Multi-season data: the noise fix that revealed a real flaw

Goal: fix the "166 matches is too few" problem from session 5 by
stacking multiple seasons. Stacked 6 complete League One seasons
(20-21 through 25-26) = 3,312 matches, 8.6x the original 386.

The promotion/relegation reality:
- NOT 24 teams across 6 seasons — 49 distinct teams. Only Lincoln and
  Burton appear in all 6 (the League One lifers). A long tail appear
  just once (Birmingham, Wrexham, Sunderland) — passing through on the
  way up or down. The data encodes each club's trajectory.
- This is WHY you can't just flat-average a team across all seasons:
  (1) unequal samples (Lincoln has 6 seasons, Birmingham 1), and
  (2) strength isn't stable across years — a 2020 team tells you little
  about the 2025 version. Old data is stale, not just thin. COVID
  season (20-21) especially distorted: empty stadiums reduced home
  advantage that year.

The method I chose (Option 1 — per-season, then pool):
- Run the full train/test pipeline INDEPENDENTLY inside each season
  (train on that season's early games, predict its late games), then
  POOL all 6 seasons' out-of-sample predictions = 996 predictions.
- Why this one: strengths stay clean (only ever from the same season,
  no staleness, no blending), but I still get 6x the calibration data.
  It doesn't make any single prediction smarter — it gives me a
  trustworthy calibration baseline. That's exactly what session 5
  said I needed. (Recency-weighting is the NEXT step, not this one —
  it needs this baseline to be tuned against.)

The payoff — two findings:
1. The session-5 "overconfidence" panic (0.2-0.3 bucket, predicted
   0.25 vs actual 0.11) was NOISE. With 159 matches instead of 35 it's
   now 0.250 vs 0.226 — nearly perfect. Lesson confirmed: never act on
   thin buckets. I was right to flag it AND right not to panic.
2. A REAL pattern emerged that 166 matches had buried: the model is
   systematically OVERCONFIDENT AT THE EXTREMES. When it predicts a
   high home-win prob, home wins LESS often (0.646 -> 0.551, n=127);
   when it predicts low, home wins MORE often (0.146 -> 0.239, n=113).
   The probabilities are too SPREAD OUT; the truth is compressed toward
   the middle. On the curve it's an S-shape bowing away from the
   diagonal at both ends.

The meta-lesson:
- More data didn't just firm up numbers — it REVEALED a real,
  systematic, fixable flaw I literally could not see at 166 matches.
  This vindicates the whole session. Signal was always there; I needed
  the sample size to see it past the noise.
- Mechanically the flaw makes sense: raw goal-average strengths don't
  account for small samples producing extreme-looking teams who then
  regress to the mean. The model takes extremes at face value.

Next session: the fix for overconfidence — shrinkage / regression to
the mean (pulling extreme strength estimates back toward average).

## Session 7 — Shrinkage: fixing the overconfidence

Goal: fix the systematic overconfidence found in session 6 (model's
probabilities too spread out, truth compressed toward the middle).

The technique — shrinkage (regression to the mean):
- Pull every team's strength estimate PARTWAY back toward the league
  average, weighted by how many games it's based on.
- Formula: shrunk = (n*team_avg + k*league_avg) / (n + k)
  where n = games played, k = "phantom league-average games" added to
  every team. Few games (small n) -> lean toward league average (don't
  trust thin data). Many games -> lean toward the team's real average.
  k controls how hard you shrink. k=0 = no shrinkage (old model).
- WHY it's the right fix: raw goal-averages take small samples at face
  value — a team that scored a lot in a few games gets tagged "strong"
  but some of that was luck, not skill. Shrinkage trims the luck.
- It's the discrete cousin of Bayesian estimation: league average is
  the "prior," each team's data updates it. Same idea used in baseball
  sabermetrics and a lot of finance for noisy estimates.

The result (k=5, ~16 training games per team):
- Reproduced session 6 exactly at k=0 first (so any change = shrinkage,
  not a code artifact). Good habit: isolate the variable.
- High-end overconfidence gap COLLAPSED:
  - 0.6-0.7 bucket: gap went from +0.095 (predicted 0.646 vs actual
    0.551) to +0.014 (0.639 vs 0.625). Nearly perfect.
  - 0.7-1.0 bucket: from +0.050 to -0.028.
- The curve pulled in toward the diagonal, most at the high end where
  the worst flaw was.
- Honest caveat: the LOW end (0.0-0.2) barely improved — still ~7-point
  gap. The underdog/away side is still a bit overconfident. Less data
  and more noise on away form. Noted, not fixed.
- Bucket counts shifted: extreme buckets shrank (top went 66->22),
  middle swelled. Correct — the model stopped making over-extreme
  claims. That's the compression working.

The DISCIPLINE lesson (most important):
- Tempting to try lots of k values and pick the prettiest. That's
  OVERFITTING TO THE TEST SET — using test data to make a modelling
  decision silently turns it into training data and inflates how good
  the model looks.
- To tune k honestly I'd need a THIRD slice: a validation set, separate
  from train and test. Try k values on validation, check the winner
  ONCE on test. I don't have that split, so the honest move is to pick
  a defensible k from reasoning and NOT tune against test.
- k=5 chosen from problem structure (~1/3 of a team's game count —
  shrinks noise without erasing real differences), NOT from peeking.
  "Why k=5?" -> "roughly a third of game count, shrinks noise without
  erasing signal, deliberately not tuned against test." Strong answer.
  "I tried 20 values and 5 won" -> weak answer even if it sounds
  thorough.

Future improvement: proper k tuning with a train/validation/test split.
Also still open: the underdog-side overconfidence, and recency-weighting
from session 6.

Next: with calibrated probabilities, start the odds-side work — overround
and margin removal (Buchdahl), to compare my probabilities vs the market.

## Session 8 — Odds-side begins: implied probabilities and the overround

First contact with the market side. Sessions 1-7 built MY probability
estimate; this extracts the MARKET's, from the bookmaker odds in the CSV.

The core conversions:
- Decimal odds -> implied probability = 1 / odds. (Odds of 2.00 imply
  a 50% chance.)
- The OVERROUND = sum of the three implied probabilities (home + draw +
  away). For real probabilities this should be 1.0 (100%). It's always
  MORE — the excess is the bookmaker's margin / vig / juice.
- A sum over 100% can't be a real probability set — it's the margin
  inflating the raw inverses. So raw 1/odds is NOT the market's true
  probability; it's inflated and has to be de-margined.

Which odds to use:
- CSV has ~20 bookmakers. Pinnacle is the sharpest (Buchdahl's whole
  conclusion), columns PSH/PSD/PSA, and PSCH/PSCD/PSCA for CLOSING.
- Use CLOSING (PSC). Why: the closing line is the final price before
  kickoff — most time to absorb all betting action, sharp money, late
  info (injuries, lineups). It's the single sharpest probability the
  market produces. "Beating the closing line" = beating PSC.
- Also present: Avg/Max across all books. Those matter LATER for the
  value hunt (compare best available price vs sharp true price). Not
  needed for establishing true probability.

The result (24-25 season, 165 matches with Pinnacle closing odds):
- Average overround: 1.0418 = a 4.18% margin.
- DISCREPANCY worth noting: Buchdahl found Pinnacle ~2.7%. Mine is
  higher. Why this is REAL, not an error: his sample was worldwide top
  divisions; League One is lower and less liquid. Lower-tier football
  carries fatter margins — less sharp money to tighten the line, more
  uncertainty. The wisdom-of-the-crowd effect is weaker where the crowd
  is smaller. Still far tighter than UK books (6-8% here).
- Margin spread is tight (1.037 to 1.060) — Pinnacle applies a fairly
  consistent margin across matches.

Where this leaves me:
- I now have the market's RAW implied probabilities (still inflated by
  the margin) AND my model's calibrated probabilities (sessions 1-7).
- They are NOT comparable yet — the market numbers are inflated. Must
  strip the margin first. And the margin is NOT spread evenly: the
  favourite-longshot bias inflates longshots more than favourites
  (Buchdahl). Naive fix (divide each by overround) ignores that.

Next session: strip the margin properly — the three Buchdahl methods
(equal / proportional-to-odds / and the bias-aware ones) — to get the
market's TRUE implied probabilities. Then the two estimates are finally
comparable = the edge hunt.
