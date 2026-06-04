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
