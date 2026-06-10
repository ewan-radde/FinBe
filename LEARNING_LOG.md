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
