# CAE-bench trial architecture & generality

How a single trial actually executes — and which parts of this design
extend to simulation domains beyond the ones we currently ship (see
[`../CASES.md`](../CASES.md) for the live list).

This doc is the level above the code: how the pieces fit, and a
first-principles read on which assumptions can travel. The hooks it used to
point at went with the self-built runner.

## Trial pipeline (left → right time)

Harbor owns this. `tools/run_harbor_trial.sh` picks a dataset, an agent
(`terminus-2` — the board's scaffold — / `claude-code` / `codex` / `oracle` /
`nop`) and a credential set; Harbor builds
the environment from `task.toml`'s `docker_image`, runs the agent in it, then
starts a **second** container for the verifier when the case declares
`environment_mode = "separate"`, and collects the paths named in `artifacts`.

```
harbor run -p <dataset> -a <agent> -o <jobs dir>
  │
  ├─ container 1: agent      reads instruction.md, drives the solver,
  │                          writes deliverables under /tmp/agent/submission
  │
  └─ container 2: verifier   re-extracts the KPIs from those deliverables and
     (network: none)         writes /logs/verifier/reward.json
```

What we own inside that is the case, the verifier library and the domain image —
not the loop. The harness layer and the two Claude Code hooks that used to sit
here were deleted with the self-built runner; `docs/infra_migration_plan.md`
records why, and the short version is that a hook was compensating for a case
defect rather than an agent one.

## Detector layer (post-stop, post-score)

The verifier (`score.py`) runs after the agent has stopped. It
computes scores, then dispatches the per-KPI score dicts through the
detector plugin layer (`lib/sim_benchmark_verifier/sim_benchmark_verifier/detectors/`).
Detectors emit two annotations onto each per-KPI entry — both are
**diagnostic-only metadata, not score components**.

| Axis | Values | Detector(s) |
|---|---|---|
| `provenance_stage` | P0_hallucination · P1_path_invalid · P2_extract_unrunnable · P3_extract_mismatch · P4_pass · spec_error | universal (single, solver-agnostic) |
| `solver_stage` | L0_input_syntax · L2_solver_crash · L3_convergence · L4_conservation · L5_physics · L5_quantitative · L6_pass · (open enum) | universal (L2/L5/L6 from KPI fields) + openfoam (L3/L4 from log scan) + future per-solver |

Two-axis split solves a v3.1 confusion: provenance failures
("paperwork") and solver failures ("physics") need different fixes.
See `SCHEMA.md §6` for the full enum + back-compat with `failure_class`.

## First-principles generality analysis

The architecture above makes specific assumptions about what a
"simulation trial" looks like. Below: which assumptions actually
generalize to other domains, and which need extension.

### Strong assumptions (built into the contract)

#### A1. Output is one or more **scalar KPIs** with a deterministic extraction rule

Currently: `result.json = {kpi: {value: <number>, source: <how to re-extract>}}`.

| Domain | Fits? | Notes |
|---|---|---|
| Combustion / battery KPIs (ignition delay, flame speed, terminal voltage) | ✅ | Native fit — the solver's own output carries scalar measures |
| CFD numerical KPIs (Cd, Cl, residual, max y+) | ✅ | Native fit |
| FEA stress / displacement at named locations | ✅ | Same shape — extract from solver's text output |
| EM frequency-response, S-parameters | ✅ | Scalar at chosen frequency |
| Robotics: collision count, path length, settling time | ✅ | Scalar from rosbag / sim log |
| **Field outputs (CFD vorticity field, FEA stress contour)** | ⚠️ | Single scalar reduction (max, integral, line-probe) fits; full field comparison needs different verifier |
| **Image outputs (camera frame from robotic sim, rendered view)** | ❌ | No scalar; would need perceptual similarity metric or LLM judge |
| **Time-series / trajectory (joint angles over time)** | ⚠️ | Reduce to scalar features (overshoot, settling time, RMS); the contract works, the case author has more upfront work |
| **Stochastic outputs (Monte Carlo, RL training reward)** | ⚠️ | Single trial returns a sample, and the tolerance band is binary — it accommodates **no** fuzz, so a sampling spread comparable to `pass_tol` scores at random. The fix is a KPI defined on the distribution (a quantile, a fixed-seed statistic), never a wider band: a band widened until both a right and a wrong answer pass has stopped measuring anything |

#### A2. Extraction is **re-runnable** by the verifier

Currently: agent declares `(value, source.path, source.extract)` →
verifier re-runs extract with file as stdin, asserts match.

| Source kind | Generalises to | Notes |
|---|---|---|
| `file_extract` (text file + UNIX pipeline) | Any text-format solver output | Most solvers; covers CFD logs, FEA text output, structured text dumps |
| `sim_run_*` (sim-cli's own record) | Anything that runs through sim-cli | Optional — most agents bypass |
| **Binary-format outputs (HDF5, VTK, ROS bag)** | Need adapter | New `source.kind` per format — e.g. `h5_extract` with HDF5 path query, `rosbag_query` with topic + time |
| **Image outputs** | Don't fit re-extraction model | Need similarity-metric verifier; outside this architecture |

#### A3. Workflow is **agent → solver → final state → verifier**

Currently: agent writes inputs, runs solver, writes `result.json`,
verifier scores once at the end.

| Workflow shape | Fits? | Notes |
|---|---|---|
| Batch CFD / FEA / kinetics (run once, score once) | ✅ | Native fit |
| Multi-step CAE (mesh → solve → post-process → judge) | ✅ | Multiple solver calls within one agent loop; verifier still scores once at end |
| **Interactive GUI (Fluent meshing workflow, COMSOL Desktop attach)** | ⚠️ | Agent does many small actions; verifier still scores final state. PreToolUse hooks could add per-step validation if needed. |
| **Closed-loop control (robot follows commanded trajectory)** | ⚠️ | KPI is final-state scalar (e.g. position error at t=T), so contract works; but the SIMULATION environment is more complex (need physics step + sensor feedback loop). Requires real-time sim platform inside the container. |
| **Multi-trial campaigns (Monte Carlo, parameter sweeps)** | ⚠️ | One trial = one sample; aggregate at harbor level via multiple trials. Doesn't break the architecture, but the per-trial score becomes statistical. |

#### A4. Time budget is **bounded per Bash call** (claude-code's 600s ceiling)

Currently: agent's bash calls each capped at ≤ 600s (claude-code
[issue #25881][1]). For longer sims, agent must use solver
checkpointing + multiple foreground calls, or `run_in_background=true`.

[1]: https://github.com/anthropics/claude-code/issues/25881

| Sim wall-time | Pattern |
|---|---|
| < 5 min | Single foreground call. Default. |
| 5 – 30 min | `run_in_background=true` + claude-code's output-polling tool. Hook auto-skips wrapping for background mode. |
| > 30 min | Solver checkpointing (`writeInterval` for OF, `RESTART_SOL=YES` for SU2) + multiple foreground calls. Each retry continues from previous endpoint. |
| > 1 day | Doesn't fit at all. The trial container itself is bounded by harbor's `agent.timeout_s × timeout_multiplier`. Architecture would need a different runtime (job queue + checkpoint orchestration). |

#### A5. Agent framework is whatever Harbor can drive

This assumption **was** "the agent framework is claude-code", and it is no longer
true. Frameworks differ in ways that reach the measurement: the live example is
the turn budget — `claude_code` exposes `max_turns`, `codex` exposes no such
flag, so a 60-turn cap applied to one and not the other inflated the codex row's
lead until every capped trial was re-run without it.

#620 settles it the other way round for the published board: **the board runs on
`terminus-2` and no other scaffold**, and `claude-code` was dropped. A row from
any other scaffold is a diagnostic or a control, and it may not be pooled into a
board aggregate — the scaffold is part of what the number measures, so two of
them under one mean is two measurements pretending to be one.

So the assumption that survives is weaker and worth stating plainly: **a task
must be answerable by any agent Harbor can drive, and any budget we impose must
be expressible for all of them.** A budget only one framework understands is not
a measurement parameter, it is a handicap on the others.

### Things that travel cleanly across simulation domains

Independent of the assumptions above:

- **Source-provenance verification.** "Re-extract from declared
  source" works for any deterministic output. New `source.kind` per
  format.
- **Two-axis failure attribution** (`provenance_stage` ×
  `solver_stage`). Provenance axis is solver-agnostic. Solver axis
  uses per-domain detector plugins; the universal stages (L0 / L2 /
  L5 / L6) apply to every solver, domain-specific stages get added
  per-detector.
- **Calibration discipline.** Real broken/healthy fixture pairs +
  TPR/FPR validation per detector. `EVIDENCE.md` template applies to
  any new domain.
- **Cost / wall / turns instrumentation.** `cost_meter.py` reads
  claude-code transcripts; agent-framework-specific but the *concept*
  (capture turn count + USD cost per trial) applies anywhere.

### Domains we'd need to extend the architecture for

Concrete examples of where the current scheme falls short:

| Domain | What breaks | Required extension |
|---|---|---|
| **Computer vision in robotics sim** | Image-output → no re-extractable scalar | New source kind: `image_metric` with reference image + similarity threshold. Verifier needs CV library. |
| **Autonomous driving sim** | Long-running real-time + closed-loop + safety-critical | Different runtime (CARLA/SUMO instead of Docker), different harness (real-time sim API vs shell), different verifier (trajectory + collision metrics). Architecturally a fork. |
| **Quantum circuit simulation** | Outputs are state-vector / density-matrix / shot histogram | New source kind: `quantum_state` with HDF5 / npz reader; KPI extracts measurement-basis probabilities. Numerical contract holds. |
| **Climate / weather** | Multi-day wall-clock, terabyte outputs | Doesn't fit container model; needs job-scheduler runtime + remote storage. |

The good news: the **first three layers (harness install, hooks,
verifier core)** generalise as long as we extend the source-kind set.
The runtime/infrastructure layer (claude-code, harbor, docker, 600s
ceiling) is the harder constraint.

## When the architecture is the right tool

Use it when:
- Output is a finite set of scalar KPIs extractable deterministically from solver artifacts
- Per-trial wall ≤ ~30 min (with checkpointing) or bounded enough for `run_in_background` polling
- Verification is rule-based, not human-judgment-based
- Solver runs in Linux (Docker) or via wine (commercial via wrapper)

Don't force it when:
- The "right answer" requires human aesthetic judgment (shape, layout, design)
- Outputs are inherently non-scalar (full fields, images, trajectories without natural reductions)
- The simulation needs real-time stimulus / closed loops outside container's reach
- Single trials take > 1 day

## References

- [`SCHEMA.md`](../SCHEMA.md) — case + verifier contract, two-axis enum
- [`../lib/sim_benchmark_verifier/EVIDENCE.md`](../lib/sim_benchmark_verifier/EVIDENCE.md) — detector calibration TPR/FPR
- [sim-proj#125 RFC (private)](https://github.com/svd-ai-lab/sim-proj/issues/125) — two-axis design discussion
