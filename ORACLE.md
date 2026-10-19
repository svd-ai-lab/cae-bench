# Oracle baseline — is the apparatus sane?

Every case ships a known-good `solution/solve.sh`. Running it through the
`oracle` agent involves no LLM and no sampling: same case, same image, same
number. That makes the oracle the instrument check — **if a reproducer's
oracle run disagrees with yours, the environment is broken, and no
agent score from that environment means anything** until it is fixed.

Scores are not recorded in this repo. They are cheap to recompute from the
artifacts, and a number pasted into a markdown file goes stale the moment a
toolchain moves. Run the oracle and read what it prints.

## Running it

```bash
# one case, through the same runner a real trial uses
harbor run -p cases/<domain>/<subdomain>/<case-id> --agent oracle -y
```

To run a case's oracle in the **two-container shape a real trial is graded
in** (agent container writes, verifier container scores, no shared state)
and keep the outcome for review:

```bash
python tools/run_oracle_into_store.py <case-dir>     # records into results-local/
python tools/run_oracle_all.py                       # sweep a domain / the catalog
```

`results-local/` is gitignored on purpose: results are data produced by a
particular machine at a particular commit, not source. If a result set is
worth keeping, attach it to a Release. The numbers on the published board
(`cae-bench.svdailab.com`) are kept a third way — as immutable snapshots
written by `tools/publish_board.py`, so a figure quoted off the page can be
checked against the snapshot it came from.

## What the number has to satisfy

These are contract thresholds, not measurements — they hold for every case
in the catalog or the case is defective:

| Run | Required | Why |
|---|---|---|
| `solve.sh` against its own `tests/` | **= 1.0** | the band is binary since #188/#193, so anything between 0 and 1 means a whole KPI group did not pass — the reference fell out of its own band, and either the case or the oracle is broken (#359) |
| a deliberately broken run | **< 0.5** | if breaking the physics doesn't move the score, the KPI isn't measuring the physics |
| the same oracle, run twice | identical | non-determinism in the oracle means the tolerance band is measuring noise |

An oracle that scores *exactly* 1.0 on a KPI whose ground truth came from
that same oracle is proving reproducibility, not correctness — see the
`gt_type` and `oracle_provenance` fields in `tests/kpis.json` for which
claim a given case is making.

## When the oracle disagrees with the reference literature

Expected, and it must be written down rather than smoothed over. A coarse
mesh, a wall-function boundary layer or a different turbulence model will
land the oracle away from the experimental value; `oracle_provenance` and
the per-KPI `T_good_source` are where that gap is justified. What is **not**
acceptable is a case whose `instruction.md` quotes the experimental value
while the grader wants the oracle value — that penalises the agent for being
right. Check both before trusting a case.

## Related

- [`SCHEMA.md`](SCHEMA.md) — the KPI + scoring contract the oracle is judged by.
- [`REPRODUCING.md`](REPRODUCING.md) — environment setup paths.
- [`CASES.md`](CASES.md) — catalog, per-case oracle status.
