# Leaderboard — how a model is scored and ranked

This file documents the **ranking method**. It deliberately holds no scores:
a leaderboard row is a measurement of one model, on one commit, in one
environment, and re-deriving it from the stored artifacts is cheap. Numbers
pasted into markdown are not.

Where results actually live:

- **`results-local/`** (gitignored) — the file-backed store written by
  `tools/bench_store.py`. Everything a review needs per trial: `reward.json`,
  `reward_detail.json`, both containers' logs, figures, indexed by
  `(run, case, label)`. Browse it with the review page (`results-local/review.html`).
- **The published board** — [cae-bench.svdailab.com](https://cae-bench.svdailab.com),
  the numbers as they stand. Each publication is an immutable snapshot under
  `results-local/board/`, written by `tools/publish_board.py` and rendered into
  the page by `tools/render_public_board.py`; `tools/refresh_public_board.sh`
  runs the whole chain and verifies the deploy. A snapshot is what makes a
  quoted figure re-checkable — the page moves, the snapshot behind it does not.
- **Releases** — if a result set is worth preserving, attach it to a Release.
  Not to the source tree.

## The scoring chain

Per trial, per case, the verifier produces one number (`/logs/verifier/reward.json`,
single key per Harbor's contract) and a full breakdown alongside it
(`reward_detail.json`, schema `reward-v3`). The chain, defined normatively in
[`SCHEMA.md`](SCHEMA.md):

```
final_score = W_KPI · kpi_score · numerics_ok      # W_META = 0 — the meta layer is diagnostic only
kpi_score   = Σ group.weight · group_score         # group weights sum to 1.0
group_score = mean(member kpi scores)
per-KPI     = source_verified · physics_pass · band_pass    # band_pass is BINARY
numerics_ok = 0 if a gated process KPI (residual, mesh count) is reported
              outside its declared limit, else 1
```

with a per-solver **artifact detector** hard-zeroing the whole case if the run
left no evidence a solver actually ran.

Two properties of this chain matter when comparing models:

- **Which tools the agent used is not scored.** The meta layer that once gated
  on driving the solver through a blessed wrapper was removed from the score in
  a while ago and now only annotates. The reason is empirical: mandating the
  wrapper measurably *lowered* scores across every model tested, hitting hardest
  the models that obeyed the instruction, while supplying domain documentation
  with no mandate raised them. Mandates measure compliance; artifacts measure work.
- **A case's ceiling is its oracle, not 1.0.** Compare a model against the
  case's own oracle run, not against a nominal perfect score.

## Aggregation and accounting rules

### The row, in one sentence

**A leaderboard row is the equal-weighted mean of the per-track mean scores, over
one task per physics family, at one trial each.** Everything else in this section
is the accounting that sentence depends on.

Three of its clauses are load-bearing, and each is a decision that was measured
rather than assumed:

| clause | why it is that way |
|---|---|
| **one task per physics family** | within-family correlation on this store is ≈ 0, so 50 operating points of one reaction carry the resolution of a handful. The representatives are named in [`tools/board_population.txt`](tools/board_population.txt) — composed once from the union of the oracle evidence, then **read, never re-derived**: a population recomputed per host has twice produced a wrong board (see that file's header). Counting directories overstates the board. |
| **per-track mean, then equal weight** | weighting by task count would let whichever physics we generated most variants of set the headline: holding every per-track mean fixed while two tracks grew from 20→50 and 11→50 moves a pooled mean by **+0.10 to +0.16** with no change in any model. Equal weight is the composition-invariant choice, and it is why the frozen set may stay uneven. |
| **at one trial each** | repeats buy ranking power, not headline precision, and the gap that matters here (0.25–0.33) is already well above the MDE at r=1. Derivation and the N×r table: [`docs/scoreboard_sizing.zh.md`](docs/scoreboard_sizing.zh.md), re-derivable with `python tools/power_analysis.py`. |

Two rules about what may be *said* about a row:

- **Ranking is a paired comparison**, because every model runs the same tasks.
  Comparing two marginal intervals instead is not conservative, it is wrong in
  both directions — on this store it called a 0.30 gap a tie.
- **A single composite is only quotable with its interval, and only across a
  named frozen set.** A number from one frozen set and a number from a later one
  never go on the same axis; that is what versioning the set is for.

```bash
cat  tools/board_population.txt                                  # THE SET the board scores
bash tools/board_sweep.sh -m <model> --dry-run                   # the exact sweep each column ran
python tools/frozen_set.py                                       # eligibility + cost, NOT the board's set
python tools/frozen_set.py --min-models 4                        # the comparable subset
python tools/plot_scoreboard.py --unified --runs ... --models ... # the row + its interval
python tools/aggregate_leaderboard.py jobs/<name>/<timestamp>/   # --details for per-case breakdown
python tools/aggregate_economics.py  jobs/<name>/<timestamp>/    # turns, wall, tokens, USD
```

Rules a published comparison must state, because they change the number:

| Term | Definition |
|---|---|
| **Completed mean** | mean over trials that finished. Flatters models that time out. |
| **Assigned mean** | mean over every assigned task, incomplete counted as 0. The honest default. |
| **Harness exception** | an agent- or runner-level exit event (wall-time cap, router fault, missing `reward.json`). **Not** automatically a zero — the verifier may still have replayable artifacts. Count them, report them, never silently drop them. |
| **Turn cap / wall-clock** | the agent used its whole budget. **The flag is a marker, not a verdict: if the verifier scored what the trial handed back, that score stands.** Six rows on this store are exactly that — a complete, correct submission from a run that then ran out of turns, one of them at 985 s, nowhere near a time limit. Zeroing those would count work that was done as work that was not. What the flag records is *cost*, and cost is published beside the bar (the budget-flagged count per model) and in `aggregate_economics.py`, not subtracted from the score. The asymmetry that justifies this is the one the **Retry storm** row already states: a budget limit is a monotone handicap — it can cost an answer, never improve one — so a score earned under it is a floor on capability. A budget-exhausted trial that produced **nothing** scores zero, because there is nothing to score; that is the case the old wording ("without finishing") meant and the implementation over-applied. |
| **Retry storm** | a trial that completed but logged ≥ 200 HTTP 429s. Its **zero** is dropped, its **non-zero score is kept.** The asymmetry is the point: throttling is a monotone handicap — it costs turns and wall-clock and can never improve an answer — so a score earned under back-pressure is a floor on capability, while a zero cannot be told apart from never reaching the endpoint. The threshold comes off the measured distribution, not a guess: at 51–200 retries the screen model still returned 24 full scores against 6 zeros; above 200, 7 of 9 were zeros. Ceiling classification is unaffected, since it requires every trial at 1.0. |
| **Which trial represents a case** | one row per (model, case). A re-score supersedes the row it was derived from; otherwise a scored trial beats an unscored one, and among scored trials the one measured under the *looser* conditions wins — an uncapped re-run beats the capped original **even when it scored worse**. That is deliberate and it is not cherry-picking: taking the higher of two real measurements would let repeats be mined for the best draw. It costs one model 0.026 on this store, in the direction of the lower number. |
| **Oracle ceiling** | the same case's deterministic `solve.sh` score, on the same image and commit. |
| **Withheld case** | a case held off the board for **every** model because a defect makes some of its rows wrong and those rows genuinely cannot be re-derived. Listed with its reason in `results-local/scoreboard_withheld.txt`, which `frozen_keys()` reads; `tools/frozen_set.py` prints what is being withheld and why. The file is normally empty — withholding is what you do when re-measuring is impossible, not when it is inconvenient. |

**"The submission is gone" is a claim to verify, not assume.** The store on any
one machine holds the rows a sweep produced, not the artifacts behind them: a
trial's submission lives on the host that ran it, and cfd trials are spread
across execution hosts by which credentials each one has. Three cases were once
withheld on the reasoning that five rows could not be re-scored — every one of
those submissions was sitting in a job directory on the execution host, and
re-scoring them there cleared all five. So before concluding a row is
unauditable, go look on the machines that ran it; `no-artifact` in
`scan_store_readiness.py` means *not in this store*, which is a much weaker
statement than it sounds.

An evaluator fix does not repair a row — re-scoring the stored submission does.
So a defect splits a case into rows that can be re-derived and rows whose
submission is not on this host, and **repairing only the first kind moves the
bias instead of removing it**: the fix can raise a score and never lower one, so
the model whose artifacts survived gets its zeros cleared while the model whose
artifacts did not keeps them. Withholding the whole case restores symmetry at
the cost of resolution, and that is the right trade — a case measured correctly
for three models and wrongly for two carries less than no case at all, because
it carries it in a direction nobody can quantify. A withheld case comes back by
being re-measured, not by being argued about.

A model comparison is only meaningful within one commit: `environment/` and
`cases/` move together, and the reproducibility coordinate is the git commit
(see [`environment/domains/VERSIONS.md`](environment/domains/VERSIONS.md)).
Rows from different commits are not the same benchmark.

## How many trials a row needs

The headline claim is a **per-model mean over the whole case set**, not a
per-case verdict, and that changes the sampling design. With near-bimodal
per-case scores (σ ≈ 0.45), 100 cases at one trial each give SE ≈ 4.5 pp on a
model's mean; the difference of two means carries SE ≈ 6.4 pp, so models more
than ~13 pp apart are already separated without repeats. **A large case set
substitutes for repeats on the ranking claim.**

So: run the full set once per model, then add one further full pass only for
models whose gap falls inside the noise. Per-case repeats are required only where
a *per-case* claim is made (see `docs/acceptance.md` L3) — buying them everywhere
triples the bill for a claim the leaderboard does not make.

Two things this design depends on:

- **Infra loss must be small.** Lost trials shrink the effective N that the SE
  above assumes. Losses over a few percent invalidate the substitution.
- **Cases are pre-screened.** Cases the cheapest available model passes easily
  are dropped from the leaderboard N before frontier tokens are spent — see the
  screening cascade in `docs/acceptance.md`. They carry no ranking signal.

## Cost accounting

`tools/cost_meter.py` reads per-trial transcripts; `aggregate_economics.py`
rolls them up. Turns and wall-clock are cross-model comparable as-is. USD is
not — it depends on the vendor's price table at the time of the run, so a cost
figure has to carry its price basis or it means nothing. Cost per case buys
score only when read next to the oracle ceiling.

## Known limits on interpretation

- **Rows record which evaluator scored them only from 2026-08-05 onward.**
  A tag does not identify content: the same image name held two different
  graders on two hosts, one of them missing the only evaluator a whole track
  scores through. Since that date each ingested row carries the fingerprint of
  the evaluator package inside the image that produced its number, and
  `tools/store_drift.py` reports the census plus any cell whose rows came from
  more than one grader. **Rows scored before that date carry a blank, and a
  blank means not recorded — never "unchanged".** They are deliberately not
  backfilled: the boundary would have to be inferred rather than read, which
  buys a label no more reliable than the guess it replaces. The board prints
  how many of the trials behind it are in that state, and the sentence
  disappears on its own once none are.

- **The one-commit rule is stated above and nothing enforces it.** The recorded
  run set is clean today — every row on it postdates its case's last band or
  runtime change — but that is a property of *which runs are on the board*, not
  of the store. Widening `--runs` reaches back into pre-change trials and looks
  strictly better while doing it: more data, each model's best row kept. On this
  store a hand-built superset moved one model +0.025 and flipped the top two.
  So the population lives in `results-local/scoreboard_runs.txt`, which the tool
  reads, and `tools/scan_store_readiness.py` carries `cross-commit` as a
  signature (issue #13).

  Not every such edit means the same thing. Raising a **reproduction budget** is
  one-directional — it can let a correct run finish, never change what a correct
  answer is — and applies per case, so every model is flagged equally. A band or
  KPI change has neither property, and is what the rule was written for.

- **Not every stored row is scorable, and the unscorable ones are invisible.**
  Run `tools/scan_store_readiness.py` before quoting a board; each of its
  signatures is a defect this store actually carried. Two are worth knowing by
  hand, because both look exactly like a model that could not do the physics:

  **A zero the instrument caused.** A run that reproduced, meshed, converged —
  and then could not have its number read out of it, or was measured against
  something the task never asked for. The loudest form is a case scoring 0.0 for
  *every* model while its own oracle scores 1.0; that is almost never a
  difficulty finding. Nine of nineteen cfd cases carried one, and the fix was
  the same each time: see the authoring rule in
  [`.claude/skills/benchmark-case-authoring`](.claude/skills/benchmark-case-authoring/SKILL.md).

  **A pass that never solved anything.** `clean_generated_artifacts` deletes
  time directories above zero so the rerun must solve again, but `0/` is an
  *input* directory and survives. A submission shipping its converged field as
  `0/U`, with a `residualControl` loose enough that one iteration declares
  convergence, turns the rerun into an echo of its own answer — measured once at
  1.0 in **0.65 s at t=1** against a 49 s oracle, with the artifact detector
  satisfied because a non-zero time directory does exist.

  **Nothing was built to stop it, on purpose.** Seeding only pays when the KPI
  can be produced without solving, and this one — `Ub/uτ` at a tabulated `Re_τ`
  — has a log-law closed form, which CLAUDE.md's analytical-shortcut section
  already calls the mark of a case chosen wrongly rather than one needing a new
  defence. Neutralising `0/` in the evaluator would take a steady-vs-transient
  flag per case and a rule for which fields to overwrite: the same per-case
  verifier surface that produced nine of this branch's eleven defects. Fix the
  KPI or drop the case.

  A scan signature was tried and removed. Thresholding rerun cost against the
  oracle's flagged two legitimate passes — a laminar channel is simply cheaper
  than its own wastefully-long reference — and caught none of the real one,
  which sits in a withheld case and so is outside the frozen set the scanner
  reads. A detector tuned to a single observation is the defect it is chasing.

- **Cases whose ground truth came from our own oracle** prove reproducibility,
  not correctness. `gt_type` in `task.toml` says which claim a case makes;
  don't aggregate the two kinds into one headline.
- **Cases that map 1:1 onto a standard tutorial geometry** can be answered
  partly from training-set memory. Operating points moved off the canonical
  grid (the combustion family) exist to defeat exactly this; the rest of the
  catalog has not all been swept.
- **Harness capability differs from model capability.** Read the trial log and
  `reward_detail.json`'s two failure axes (`provenance_stage` × `solver_stage`)
  before attributing a zero to the model.
