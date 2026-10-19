# Reproducing a run

Three paths, in order of increasing cost. All three assume Docker and
Python 3.12+.

Start with the oracle, not with a model. It is deterministic, needs no API
key, and tells you whether your environment is sane before any score means
anything — see [`ORACLE.md`](ORACLE.md).

## Path A — pull the prebuilt image from GHCR (fast)

**Only a case that ships an `environment/Dockerfile` has a prebuilt image**, and
most do not: a case names a domain image and usually uses it unmodified, so its
`environment/` holds the fixtures the runner uploads rather than a build
context. For those cases there is nothing to pull and Path B is the only path.
No case in the live tracks ships an overlay today, so Path A is currently empty;
it stays documented because the moment one does, this is how you get it.

Where an image does exist it is published at
`ghcr.io/svd-ai-lab/sim-benchmark/<domain>-<subdomain>-<case-id>` — the case path
with `/` replaced by `-` — under two tags: `latest`, and `env-<digest>`, which
changes exactly when that case's `environment/` changes and therefore names one
immutable build. That package namespace is the project's old name and is staying
put: an image path is an identifier, and moving it would only invalidate every
digest already pulled.

The published set is reconciled against the repository, not accumulated: on each
push and once a week, `.github/workflows/push-case-images.yaml` asks the registry
which required images are absent, builds those, and then fails the run if any is
still missing afterwards. "The image for this case never got pushed" is a red run
rather than a stale pull three weeks later.

```bash
uv tool install harbor
git clone https://github.com/svd-ai-lab/cae-bench && cd cae-bench

# If the GHCR package is still private, authenticate once:
# echo $GITHUB_TOKEN | docker login ghcr.io -u <username> --password-stdin

# --ek docker_image=... tells Harbor to pull prebuilt instead of building.
harbor run -p cases/<domain>/<subdomain>/<case-id> \
    --agent oracle \
    --ek docker_image=ghcr.io/svd-ai-lab/sim-benchmark/<domain>-<subdomain>-<case-id>:latest
```

## Path B — build from source (from-scratch verification)

```bash
# (CN dev hosts only — see the registry-override section below)
docker pull docker.1panel.live/opencfd/openfoam-default:2412
docker tag  docker.1panel.live/opencfd/openfoam-default:2412 opencfd/openfoam-default:2412

harbor run -p cases/cfd/fluids/taylor_green_vortex_2d_re100 --agent oracle
```

Path A and Path B produce equivalent runs because every layer input is
pinned. Differences confined to the floating-point tail are expected and
absorbed by the KPI tolerance band.

A case names its environment; it never installs a runtime of its own. What
version of which solver a domain image ships is recorded in
[`environment/domains/VERSIONS.md`](environment/domains/VERSIONS.md), and the
reproducibility coordinate is the git commit — `environment/` and `cases/`
move together.

## Path C — run a real LLM agent

```bash
export OPENAI_API_KEY=sk-...        # any OpenAI-compatible endpoint

harbor run -p cases/cfd/fluids/taylor_green_vortex_2d_re100 \
    -a terminus-2 -m <model-id>
```

`terminus-2` is the scaffold the published board runs on, so a row produced any
other way is a diagnostic rather than a board cell. Aggregate the resulting job
directories with `python tools/aggregate_leaderboard.py <dirs>` — the accounting
rules that make two runs comparable are in [`LEADERBOARD.md`](LEADERBOARD.md).

## Regional mirrors / registry overrides

Our `docker-compose.yaml` defaults to `docker.1panel.live` (a CN mirror of
Docker Hub). Outside CN, override to `docker.io`:

```bash
# Edit cases/_template/environment/docker-compose.yaml (and any case's
# environment/docker-compose.yaml) to set:
#     BASE_REGISTRY: docker.io
# Or pass it per-run:
harbor run --ek build.args.BASE_REGISTRY=docker.io ...
```
