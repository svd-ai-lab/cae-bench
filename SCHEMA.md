# SCHEMA — CAE-bench public v0

This file describes the case contract for the public industrial simulation
agent benchmark. It is intentionally solver-neutral: OpenFOAM, Cantera,
PyBaMM, CalculiX and future solver domains use the same Harbor task shape,
result contract, and KPI grader.

## 1. Case Layout

Each case lives at:

```text
cases/<domain>/<subdomain>/<case_id>/
├── task.toml
├── instruction.md
├── environment/
│   └── Dockerfile
├── solution/
│   └── solve.sh
└── tests/
    ├── test.sh
    └── kpis.json
```

When `oracle_status = "available"`, `solution/solve.sh` is the oracle entry
point. `tests/test.sh` runs the deterministic verifier and writes Harbor reward
files under `/logs/verifier`. Case-specific helper files are allowed.
`oracle_status = "deferred"` cases may omit `solution/solve.sh` while the
no-token oracle is still being built; this does not by itself make the task a
draft.

## 2. Harbor Task Metadata

New cases use Harbor schema version `1.3`. Version `1.1` remains valid and is
what most of the historical catalog is on; the two differ in the runtime block's
field names, and `tools/lint_case.py` enforces whichever set matches the file's
own `schema_version`. Do not mix them in one file.

```toml
schema_version = "1.3"
artifacts = ["/tmp/agent/submission"]   # ≥1 absolute path, 1.3 only

[task]
name        = "sim-benchmark/<case-slug>"   # exactly org/name; the namespace is a
                                            # fixed literal — a task id, not the
                                            # project's name. Do not rebrand it.
description = "One-line task description"
authors     = [{name = "sim-benchmark"}]
keywords    = ["solver", "topic"]

[environment]
docker_image = "sim-benchmark-<domain>-fullstack:latest"   # the domain image
cpus         = 2
memory_mb    = 4096
storage_mb   = 4096
gpus         = 0
network_mode = "public"                  # public | no-network | allowlist
                                         # `public` is what every live case
                                         # declares, and copying anything else
                                         # into a new track breaks it -- see
                                         # the note under this block

[agent]
timeout_sec = 1800
user        = "agent"

[verifier]
timeout_sec      = 1020
user             = "agent"
environment_mode = "separate"            # separate ⇒ [verifier.environment] +
                                         # tests/Dockerfile are required

[verifier.environment]
cpus         = 2
memory_mb    = 4096
storage_mb   = 4096
gpus         = 0
network_mode = "public"

[metadata.sim]
task_id         = "<dash-slug>"   # the solver-neutral task; shared by every port
solver          = "openfoam"      # needs a matching detectors/<solver>.py, or
                                  # "neutral" to opt out of the artifact gate
source_type     = "vv_standard"   # what kind of thing the reference is
source_channel  = "workshop-vv"   # which demand channel found it — a different axis
source_citation = "Human-readable source"
source_url      = "https://example.com"
difficulty_tier = "S"
gt_type         = "high-fidelity-solver"
tags            = ["solver-neutral", "scalar-kpi"]
release_status  = "public_runnable"
oracle_status   = "available"
score_template  = "measurement"
leakage_risk    = 2
capability_target = "postprocess"
prototype_origin = "cantera:samples/python/onedim/adiabatic_flame.py"
                                  # the published example this case is a variant of —
                                  # <tool>:<path> for a first-party one, a URL or other
                                  # locatable reference for a third-party one. Omit when
                                  # the collision check found nothing. Required together
                                  # with prototype_delta, and required on every
                                  # source_type = "tutorial" case — see §7
prototype_delta = "phi/T/P moved off the sampled point; ..."
```

**`network_mode = "public"` is a deliberate setting, not a default nobody
tightened.** The agent runs *inside* the container, so any other mode routes its
own model endpoint through the egress sidecar and the trial dies at turn 1 with
an empty submission and a 0.0 — a whole track once spent its entire life at zero
coverage that way, from one value copied out of a template. What keeps the open
network from being a hole is the *operating point*: the cases are posed away
from any published table, so there is nothing to look up. A case may not claim a
network restriction the environment does not impose either; `lint_case.py` fails
an `instruction.md` that does.

The 1.1 runtime block, for reading existing cases: `[environment]` takes
`memory_gb` + `internet_access` instead of `memory_mb`/`storage_mb`/`gpus`/
`network_mode`, and `[agent]`/`[verifier]` take `timeout_s` + `execution_user`
instead of `timeout_sec` + `user`.

`docker_image` may point to a shared prebuilt image for fast iteration. Cases
that build their own image must still keep the same agent/verifier/result
contract.

Required fields are enforced by `tools/lint_case.py`.

## 3. KPI Spec

KPI specs live in `tests/kpis.json` and use `schema_version:
"neutral-v0.3"`.

```json
{
  "schema_version": "neutral-v0.3",
  "case_id": "rc_highpass_ac",
  "source": {
    "primary_url": "https://example.com",
    "citation": "Reference or oracle provenance"
  },
  "kpi_groups": {
    "setup": { "weight": 0.10 },
    "outputs": { "weight": 0.90 }
  },
  "kpis": {
    "sim_completed": {
      "group": "setup",
      "shape": "scalar",
      "description": "1 if the solver run completed, else 0",
      "gt_value": 1,
      "pass_tol": 0.5,
      "gross_error_tol": 0.5,
      "physics_min": 0,
      "physics_max": 1,
      "pass_tol_source": "Binary flag"
    }
  }
}
```

Rules:

- `kpi_groups` weights must sum to `1.0`.
- Every KPI must reference an existing group.
- Every group with positive weight must contain at least one KPI. Empty groups
  are invalid case schema, not a valid way to reserve future score mass.
- Current verifier support is scalar KPIs only.
- Missing KPIs in `/tmp/agent/result.json` score zero.

#### `pass_tol` and `gross_error_tol` — the tolerance band

**Both are ABSOLUTE tolerances carried in the KPI's own unit**, not fractions
and not percentages. A KPI whose `gt_value` is an ignition delay in
milliseconds carries a `pass_tol` in milliseconds. This is worth stating
because the field it replaced, `T_good`, was never defined anywhere in this
repository: the `T` is `tolerance`, but on the combustion track it sits in the
same file as a `gt_value` in ms next to a CSV column named `T_K`, and it was
read as a temperature more than once.

| field | in the score? | meaning |
|---|---|---|
| `pass_tol` | **yes** | the value scores 1 when `\|pred − gt_value\| ≤ pass_tol`, and 0 otherwise |
| `gross_error_tol` | **no** | diagnostic only. Separates "computed the wrong number" from "not in the right postcode" for failure attribution, and is what `L5_quantitative`'s `gross_error` flag is measured against |

**Scoring inside the band is binary. There is no partial credit and no decay.**
The score answers whether a computed result is right, and a number that is
wrong does not become less wrong by being wrong by less: an ignition delay 10%
out means the mechanism, the reactor form or the peak extraction was wrong.
Reward density for anyone doing shaping comes from the continuous
`absolute_error` that every evaluator writes into `reward_detail.json`, and
from the multi-KPI and process-gate structure around the band — never from
decay inside one KPI.

`T_good` / `T_bad` / `T_good_basis` / `T_bad_basis` are the historical
spellings. The verifier still reads them so that a stored trial and the
unmigrated `cases/_phase2/` tracks keep scoring; `tools/lint_case.py` rejects
them in a live case, and the fallback is removed one release after the
migration.

### `oracle_provenance` — where the reference value came from

A sibling of `kpis` in `tests/kpis.json`. It records how a case's `gt_value`
was obtained, in enough detail that someone on another machine can get the same
number. It is not scored and the verifier never reads it; it is what makes a
tolerance band auditable rather than asserted.

```json
"oracle_provenance": {
  "method": "constant-current discharge to the voltage cut-off, integrated from the reproduced current and voltage trace",
  "pybamm_version": "26.7.1.0",
  "variants": { "baseline": { "value": 1.88456 }, "tight": { "value": 1.88456 } },
  "spread_rel": 0.0,
  "pass_tol_basis": "..."
}
```

| field | meaning |
|---|---|
| `method` | **The canonical spelling.** How the reference value was produced, specific enough to reproduce elsewhere — the model, the operating point, and the quantity actually integrated or sampled. |
| `<solver>_version` | e.g. `cantera_version`, `pybamm_version`, `openfoam_version`, `calculix_version`. **Must match that image's row in `environment/domains/VERSIONS.md`.** A case never pins a toolchain version itself; this records which one the reference was computed under. |
| `variants` / `spread_abs` / `spread_rel` | The same reference recomputed under different solver settings or grid levels, and the spread across them. This is what the band's floor is built from — narrower than the spread practitioners disagree by is scoring noise. |
| `pass_tol_basis` / `gross_error_tol_basis` | Why the band has the width it has. |

`extract_method` is a **historical spelling of `method`**, still carried by the
165 `kpis.json` files under `cases/_phase2/`. The linter accepts both so those
tracks keep passing, but **every new case uses `method`**; the old spelling is
not being propagated and is not an alternative.

Where the reference is a published value rather than something this repository
computed, `reference` or `gt_provenance` may stand in for `method` — the
completeness check treats any of the three as satisfying "say where this number
came from".

### Which KPIs may carry weight

Normative standard: [`docs/acceptance.md`](docs/acceptance.md)
"KPI-robustness standard". Summarised, because it is the rule most often broken:

- **Scorable:** analytical / 0-D / eigenvalue quantities, integral and conserved
  quantities, non-dimensional coefficients, and model-dependent quantities —
  with the model named in the instruction, because a band is not how a task pays
  for a choice it left open.
- **Diagnostic only (weight 0):** spatial extrema, quantities in the
  neighbourhood of a geometric or stress singularity, gradients. A `max` / `min`
  in the *name* is not the test — sensitivity to discretisation is. A
  time-series maximum from a 0-D model is scorable; a peak stress at a
  concentration is not.
- **`pass_tol` band:** a flat 5% of `gt_value` on every live track — the
  generated ones since #188, `cfd` since #190. The band is there to catch a
  wrong *configuration* — the nearest measured wrong configuration on the
  combustion track lands ~68% away — not to grade numerical precision, and it is
  still bounded by two checks: a perfectly-executed run must be able to score
  full marks, and no value recallable from the literature may sit inside the
  band. A case failing both needs a different quantity or operating point, not a
  different band. Two KPIs deviate and say so in their own `pass_tol_source`;
  [`docs/acceptance.md`](docs/acceptance.md) names them and is the authority.
- **Process KPIs** (`final_residual_*`, `mesh_cell_count`, `y_plus_max`,
  `max_non_orthogonality`) are threshold judgements, not weighted outputs. A
  process KPI opts into the `numerics_ok` gate by declaring a one-sided limit
  alongside its diagnostic fields:

  ```json
  "final_residual_U": { "group": "diagnostic", ..., "gate": {"max": 0.05} },
  "mesh_cell_count":  { "group": "diagnostic", ..., "gate": {"min": 10000} }
  ```

  `final_score = kpi_score · numerics_ok`, where `numerics_ok` is 0 if any gated
  KPI is present, source-verified, and outside its limit — and 1 otherwise,
  including when the signal is absent. Limits are stated by the case; nothing is
  inferred from `gross_error_tol` or `physics_min`, and a name is never matched.

## 4. Scoring Templates

Cases should use one of these templates unless a PR explains why a custom
template is necessary.

| Template | Use | Groups |
|---|---|---|
| `measurement` | Ordinary simulation plus KPI measurement | `setup 0.10`, `outputs 0.90` |
| `numerical` | Cases that explicitly evaluate convergence, residuals, or numerical stability | `setup 0.10`, `numerical 0.15`, `outputs 0.75` |
| `workflow` | GUI, export, or multi-step process tasks | `setup 0.15`, `process 0.25`, `outputs 0.60` |

The template is a scoring design choice, not a solver-wide default. Two cases
using the same solver may legitimately use different templates if they test
different failure modes.

## 5. Agent Result Contract

The agent must write `/tmp/agent/result.json`. The canonical shape is:

```json
{
  "pressure_drop": {
    "value": 42.0,
    "source": {
      "kind": "file_extract",
      "path": "/root/case/out.log",
      "extract": "python3 /root/case/extract_dp.py /root/case/out.log"
    }
  }
}
```

Each top-level key must match a KPI in `tests/kpis.json`.

Supported source kinds:

- `file_extract`: rerun an extraction command against a file produced by the
  agent or solver.
- `sim_run_stdout`: extract from a recorded sim-cli run stdout.
- `sim_run_kpi`: extract from structured parsed output recorded by sim-cli.

Bare numeric answers are not enough. If the solver fails, the agent may still
report failure-oriented KPIs, but each value must point to a verifiable log or
artifact source.

Do not hand-write, edit, or fabricate solver logs, result or field files,
or run-history records to satisfy provenance. The claimed source must be an
artifact produced by the actual run or by deterministic post-processing of that
run.

## 6. Reward Files

The verifier writes:

```json
// reward.json
{"score": 1.0}
```

`reward.json` has exactly one numeric key because Harbor expects a single
aggregate metric.

The verifier also writes `reward_detail.json` with:

- `schema_version` (currently `reward-v3.2`)
- diagnostic `meta_score` and `meta_detail`
- `kpi_score`
- per-group and per-KPI scoring detail (each per-KPI entry carries
  `solver_stage` + `provenance_stage` + legacy `failure_class` — see below)
- `kpi_detail.solver_stage_counts` and `kpi_detail.provenance_stage_counts`
  — distributions of each axis across the case's KPIs
- `final_score`

### Two-axis failure classification (per-KPI diagnostic layer)

Schema `reward-v3.2` carries two orthogonal axes per KPI, both
deterministic. Inspired by ccl-evaluator's L0–L6 design (see
sim-proj #125 RFC). Each axis answers a different question.

**`provenance_stage`** — solver-agnostic. Did the agent honestly record
what they got?

| Value | Meaning |
|---|---|
| `P0_hallucination` | KPI absent / claim has no source / unknown `source.kind` |
| `P1_path_invalid` | `source.path` missing / not absolute / file doesn't exist |
| `P2_extract_unrunnable` | `source.extract` failed to run (sandbox reject, non-zero exit, timeout) |
| `P3_extract_mismatch` | extract runs fine, extracted value differs from claim |
| `P4_pass` | extract reproduces the claim |
| `spec_error` | case author's `kpis.json` is malformed (not the agent's fault) |

**`solver_stage`** — solver-agnostic axis, solver-specific detector.
How far did the simulation make it? `null` when the verifier doesn't
have enough signal (cascading provenance failure, or detector for that
solver hasn't been written yet).

| Value | Meaning |
|---|---|
| `L0_input_syntax` | input file doesn't parse (per-solver detector) |
| `L1_input_semantics` | input not well-posed / contradicts intent |
| `L2_solver_crash` | solver started then died, or refused to start |
| `L3_convergence` | solver ran but did not converge |
| `L4_conservation` | converged but conservation broken (where applicable) |
| `L5_physics` | value outside `[physics_min, physics_max]` |
| `L5_quantitative` | value is inside the physics range but outside `pass_tol`. The entry's `gross_error` flag says whether it is also past `gross_error_tol` |
| `L6_pass` | nothing wrong on this axis |

Phase 1 populates `L5_physics` / `L5_quantitative` / `L6_pass` from
data the verifier already has. **Phase 2** (current commit) adds
`L2_solver_crash` attribution from `sim --json logs` records: when
provenance failed AND every sim run on the trial reports failure
(`ok=False` or non-zero exit), we emit `L2_solver_crash`. Records are
now preserved in `meta_detail.records` (previously popped) so the
classifier can run post-hoc on saved `reward_detail.json`.

`L0_input_syntax` / `L1_input_semantics` / `L3_convergence` /
`L4_conservation` still require solver-specific log analysis; those
remain `null` until phases 3–4 (issue #4) land.

**Why two axes.** A KPI that passes provenance but fails physics is a
model-capability problem; a KPI that passes physics but fails
provenance is a paperwork / SKILL-prompt problem. Different fixes,
different routing. A flat enum can't represent the cross-product —
`(L6_pass, P3_extract_mismatch)` is the most interesting analytical
cell ("simulated correctly but recorded badly").

**Backward compatibility.** The legacy v3.1 `failure_class` field is
still emitted on each per-KPI entry (mapped from the two-axis values),
so reward-v3.1 readers continue to work. Trial-level classes
(`wall_time`, `turn_cap`, `infra`) are NOT on either axis — they need
the harness transcript, populated post-hoc by an aggregator
(sim-proj #121 follow-up).

The final public score is KPI-only:

```text
final_score = kpi_score
```

sim-cli execution metadata is diagnostic-only. It may help explain how the
agent solved the task, but it must not be required for score eligibility.

## 7. Public Case Classification

Any case may be public, but it must be classified so
readers can separate runnable leaderboard tasks from draft task definitions and
high-leakage classics.

Classification fields live in `[metadata.sim]`.

| Field | Allowed values | Meaning |
|---|---|---|
| `release_status` | `public_runnable`, `public_draft`, `hidden_eval` | Whether the task/verifier is public and leaderboard-runnable now |
| `oracle_status` | `available`, `deferred`, `not_applicable` | Whether a no-token oracle solution is present |
| `score_template` | `measurement`, `numerical`, `workflow` | Which standard KPI group template the case uses |
| `leakage_risk` | integer `0..3` | 0 = novel/low leakage, 3 = very classic/high leakage |
| `capability_target` | `setup`, `solver_execution`, `numerical`, `postprocess`, `debugging`, `physics` | Primary capability the case is meant to stress |
| `source_type` | `vv_standard`, `paper`, `tutorial`, `novel_variant`, `forum`, `github`, `workshop`, `standard` | What kind of thing the reference is |
| `source_channel` | the directory names under `docs/demand_sources/records/` | Which demand channel the case was found through |
| `prototype_origin` | `<tool>:<path/in/the/example/set>` for a first-party example, **or** a URL / locatable reference for a published third-party one | Which published example this case is a variant of |
| `prototype_delta` | free text | What this case moved off that example. Required whenever `prototype_origin` is set |

Recommended interpretation:

- `public_runnable`: has task/verifier assets and can be included in the
  public leaderboard.
- `public_draft`: task definition is public, but it lacks enough runnable
  assets for the main leaderboard.
- `hidden_eval`: reserved for future private, license-gated commercial solver
  cases.

**`source_type` and `source_channel` are two axes, not two spellings.** A case
derived from a toolchain's own published example has `source_type = "tutorial"`
(a tutorial is what the reference is) and `source_channel = "tutorial-variant"`
(that is the channel it was found through). `tutorial_variant` is therefore not
a `source_type` value, and the linter rejects it; the demand record keeps
`source.type: tutorial_variant` on its own side.

**What decides leaderboard eligibility is perturbation, not `source_type`.** A
tutorial-derived case whose operating point has been moved off the published
one — so the answer is only reachable by running it — is eligible. One left on
the published point is `leakage_risk = 3`, and the linter refuses to let that
coexist with `release_status = "public_runnable"` or `"hidden_eval"`: a case
whose answer is recallable verbatim cannot also be scored. The perturbation is
evidenced by `prototype_delta` (and by the track README where a whole family
shares one design), never by the author asserting a low `leakage_risk`.

**`prototype_origin` and `prototype_delta` exist to evidence that perturbation,
so they are a pair or they are nothing.** The origin names the published example
the case descends from; the delta says what the case moved off it. Naming the
example without the delta reads as a confession with no defence — it says the
setup collides with something a model has seen and leaves unanswered whether the
collision was perturbed away — and a delta measured against nothing cannot be
checked at all. `tools/lint_case.py` therefore rejects either half on its own.

**The origin takes two forms, because what has to be locatable is the example,
not its publisher.** `<tool>:<path/in/the/example/set>` when the prototype
belongs to the toolchain's own published set
(`"cantera:samples/python/onedim/adiabatic_flame.py"`); a URL or any other
locatable reference when it belongs to somebody else
(`cases/cfd/fluids/channel_developing_entry` descends from a FOSSEE
spoken-tutorial and cites its page). Both are equally a leak risk: what a model
has been trained on is the published web, and a third-party tutorial is not
crawled less than a first-party one. **What is widened here is who published the
prototype, never whether a case that has one must record it** — a case with no
prototype omits both fields, and that silence is a claim the collision check was
run and found nothing.

Because `source_type = "tutorial"` says the reference *is* a tutorial, a case
carrying that value has a prototype by construction, and `tools/lint_case.py`
requires both fields on it. That rule was previously unkeyable: while the origin
admitted only a first-party `<tool>:<path>`, a case derived from a third-party
tutorial had no value it could legally write, and the rule would have failed a
correct case.

`oracle_status` is separate from `release_status`. A case can be a runnable
public benchmark task with a deterministic verifier even when its no-token
oracle is still deferred.

The public catalog is `CASES.md`. It should list every public case, including
drafts, instead of silently hiding unfinished tasks.

## 8. Lint and Verification

Useful local checks:

```bash
python tools/lint_case.py cases/cfd/fluids/taylor_green_vortex_2d_re100
python -m pytest lib/sim_benchmark_verifier/tests
harbor run -p cases/cfd/fluids --agent oracle -i taylor_green_vortex_2d_re100
```

On Windows with Docker Desktop, set:

```powershell
$env:DOCKER_HOST='npipe:////./pipe/docker_engine'
$env:PYTHONUTF8='1'
$env:PYTHONIOENCODING='utf-8'
```
