# Experiment — does the sim ecosystem affect the agent's CAE benchmark score?

> **Historical snapshot, not maintained.** This is a one-off experiment from the v17-v19 era, run on the self-built runner that no longer exists.
> Kept for the reasoning; do not read it as current state. Its arms ran on
> LTspice circuit cases, and circuit simulation (EDA) has been outside
> CAE-bench's scope since 2026-10-19.

**Status**: deferred from the public v0 launch path.

This document is retained as an appendix design note. The public benchmark is
now framed as an **industrial simulation agent benchmark**, not as a proof that
sim-cli improves scores. sim-cli usage is still useful diagnostic data, but it
is not part of the headline score and should not block the first leaderboard.

Use this experiment later if we specifically want an internal attribution study
of parser library vs launcher vs auto-loaded skills. Do not optimize the public
case set around this ablation right now.

## What we're measuring

Whether shipping the **sim ecosystem** (sim-cli launcher + per-solver
plugins + auto-loaded SKILL.md skills) materially helps an LLM agent
solve CAE benchmark tasks, and **which layer matters how much**.

## Design — 2-level 4-arm factorial (each layer adds exactly one thing)

| arm | tag | adds | tests when compared to previous |
|---|---|---|---|
| **L1 bare** | `:bare` | (baseline — wine + LTspice + Claude Code + ccr + verifier) | — |
| **L2 lib** | `:lib` | + `sim-ltspice` (Python parser library) | "does having a Python parser help?" |
| **L3 launcher** | `:launcher` | + `sim-cli` core + `sim-plugin-ltspice` (driver via entry-point + diagnose auto-emission) | "does the uniform launcher + run history + diagnose help, beyond just having the parser?" |
| **L4 full** | `:full` | + `sim-skills` mounted into Claude Code's skill discovery | "does auto-loading SKILL.md help, beyond just having sim-cli?" |

Adjacent comparisons isolate single layers. **L4 vs L1 = total ecosystem value**.

## Why 4 arms

A 2-arm design (with-sim vs without-sim) tells you the total value but
can't attribute it. A 4-arm factorial tells you **which sub-component**
is doing the work — which is what matters for engineering decisions
(do we double down on SKILL.md? on sim-cli affordances? on the parser
lib?).

## Container architecture — single multi-stage Dockerfile

`environment/wine-base-multistage/Dockerfile` produces all 4 tags from
ONE Dockerfile. Each stage `FROM`s the previous; impossible for the
inner stages to drift from the outer (a class of confound eliminated by
construction).

```
ubuntu:22.04
  + wine + LTspice + Claude Code + ccr + verifier  ← bare
  + pip install /opt/sim-ltspice                    ← lib
  + pip install /opt/sim-cli + /opt/sim-plugin-ltspice ← launcher
  + COPY sim-skills/ → /opt/sim-skills/             ← full
```

Build all 4 tags:

```bash
for tgt in bare lib launcher full; do
  docker build --target=$tgt -t sim-benchmark-wine-base:$tgt \
    -f environment/wine-base-multistage/Dockerfile .
done
```

## Fairness — three things kept constant across arms

### 1. Same instruction.md (byte-identical)

`tools/v19_uniform_discovery_section.py` rewrote every circuits
instruction.md's "Environment" section to the same uniform discovery
cue: "introspect the container before starting" with `command -v sim`,
`pip list`, `ls ~/.claude/skills` etc. Each arm sees the same prompt;
each arm's commands return truthful facts about its own container.

This is **mechanism C** (introspection) — the only design that doesn't
have prompt confounds AND doesn't artificially announce L3/L4 tools to
L1/L2 (which would erase the L4-vs-L3 auto-load delta we want to
measure).

### 2. Same hooks (Stop schema-check + PreToolUse Bash-timeout)

Both hooks live in the verifier package and install via
`agent_harness._install_hooks` regardless of which container the agent
runs in. The Stop hook is sim-aware (P0-2): when `sim` is not on PATH
(L1/L2), the run_id existence check is skipped AND the hook explicitly
tells the agent to switch from `sim_run_*` to `file_extract` source
kind. So L1/L2 agents get fair recovery feedback if they copy a sim-cli
worked example by mistake.

### 3. Same KPI scoring (KPI-only since 2026-04-29)

`W_META = 0`, `W_KPI = 1`. The meta-gate ("did agent use `sim run`?")
is now diagnostic-only — it lands in `reward_detail.json` for post-hoc
analysis but doesn't gate the final score. Without this rework, the
L1/L2 arms would score 0 by construction (no sim-cli history → meta=0)
and the experiment would be invalid. See
`lib/sim_benchmark_verifier/sim_benchmark_verifier/score.py` docstring.

## Discovery surface per arm — what's "knowable"

| signal | L1 bare | L2 lib | L3 launcher | L4 full |
|---|:---:|:---:|:---:|:---:|
| `command -v wine-ltspice` | ✓ | ✓ | ✓ | ✓ |
| `command -v sim` | (empty) | (empty) | ✓ | ✓ |
| `pip list \| grep sim_ltspice` | (empty) | ✓ | (empty: plugin uses bundled lib) | (empty) |
| `pip list \| grep sim_cli` | (empty) | (empty) | ✓ | ✓ |
| `pip list \| grep sim_plugin_ltspice` | (empty) | (empty) | ✓ | ✓ |
| `ls ~/.claude/skills` | (empty) | (empty) | (empty) | `sim-cli/  ltspice/` |
| **SKILL.md auto-loaded into system context** | (no) | (no) | (no) | **✓** |

The L4-only auto-load is the key claim of sim-skills. Adding an
explicit "you have sim-cli" announcement to L3's container would
neutralise this — so we deliberately don't.

## Smoke test — 3 cases, all 4 arms, 12 trials total

```bash
# Build all 4 tags (one-time)
for tgt in bare lib launcher full; do docker build --target=$tgt ... ; done

# Run each arm (swap → run → swap)
python tools/swap_base_image.py --to bare
harbor run --config configs/v19a-3case-smoke-bare.yaml

python tools/swap_base_image.py --to lib
harbor run --config configs/v19b-3case-smoke-lib.yaml

python tools/swap_base_image.py --to launcher
harbor run --config configs/v19c-3case-smoke-launcher.yaml

python tools/swap_base_image.py --to full
harbor run --config configs/v19d-3case-smoke-full.yaml
```

3 cases chosen to span LTspice's `.meas` patterns:
- `rc_lowpass_ac` — AC sweep with `.meas FIND/WHEN`. S-tier — even L1 should be able to pass
- `rc_pulse_response` — transient with `.meas TRIG/TARG/PARAM`. M-tier — tests SKILL.md value on a known LTspice "trick" pattern
- `rlc_step_underdamped` — transient with windowed `.meas MAX/AVG`. M-tier — tests window-picking tribal knowledge

After smoke, scale to all 20 circuits cases × n_repeats=3 for statistical power.

## Reading the result

For each (arm, case) pair, `reward.json` carries `score`. Aggregate:

```
mean_arm   = mean over 3 cases
delta_2_1  = mean_lib - mean_bare           # value of parser lib
delta_3_2  = mean_launcher - mean_lib       # value of launcher (+diagnose)
delta_4_3  = mean_full - mean_launcher      # value of SKILL.md auto-load
delta_4_1  = mean_full - mean_bare          # total ecosystem value
```

Also useful — per-KPI source-kind distribution from `reward_detail.json`:
- L1 agent should use 100% `file_extract` (forced)
- L4 agent's choice of `file_extract` vs `sim_run_*` is informative
  about which path the SKILL.md teaches them to prefer

## How a "win for sim-cli" looks

- `mean_full ≫ mean_bare` (e.g. 0.80 vs 0.30)
- Most of the gap concentrated in `delta_4_3` (SKILL.md auto-load) or
  `delta_3_2` (launcher) — that tells you which layer matters
- L4 agents predominantly use `sim_run_*` source kinds (the path the
  skill teaches), L1 agents use `file_extract` (the only path available)

## How a "null result" looks

- `mean_full ≈ mean_bare` — every arm hits the same ceiling
- Agents in all arms route to `file_extract` (i.e. ignore sim-cli
  even when present)
- That would mean: sim-cli's value is ergonomic only, not score-affecting
  for this workload. Worth investigating per-case.

## Caveats

- **One model only** (MiniMax-M2.5-highspeed). Cross-model replication
  is plausible follow-up.
- **Smoke is n=1 per (arm, case)** — fine for sanity-checking the
  experimental setup; full study should use n_repeats ≥ 3 per cell
  for statistical power on adjacent-arm deltas.
- **LTspice only** — the `:bare`-equivalent for OpenFOAM would be a
  separate Dockerfile track; deferred.
- **`sim-plugin-ltspice` v0.2.1+ required** — the v0.2.1 release does
  not have our diagnose.py; the L3/L4 arms need commit ≥ `98d256a`
  (the post-port HEAD) for the diagnose data to land in
  `parsed_output["diagnostics"]`. Pin in the build context.

## Background — the rework history

- **2026-04-29 morning**: original v19 designed as 2-arm
  (with-sim/without-sim) on two separate Dockerfiles
  (`wine-base/` + `wine-base-nosim/`).
- **Same day, after first round of design review**: 3 P0 fixes
  (audit + sim-aware Stop hook + worked-example file_extract primary).
- **Same day, evening**: redesigned as 4-arm factorial after deeper
  attribution discussion. Multi-stage Dockerfile replaces both legacy
  wine-base and wine-base-nosim. swap_base_image.py extended to 4 modes.
  diagnose.py ported from sim-cli monolith (now deprecated) to
  sim-plugin-ltspice (commit `98d256a`).

## Pre-flight checklist before running smoke

- [ ] All 4 image tags built (`docker images | grep sim-benchmark-wine-base`)
- [ ] `tools/swap_base_image.py --status` shows expected layer for each run
- [ ] `command -v sim` returns expected (empty for bare/lib, path for launcher/full)
- [ ] `ls ~/.claude/skills` in a `:full` container shows `sim-cli` + `ltspice`
- [ ] `sim --json logs` in `:launcher` returns valid JSON (post-build sanity)
