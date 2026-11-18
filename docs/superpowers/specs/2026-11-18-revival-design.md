# kube-ops-view Revival — Findings & Design

Date: 2026-11-18
Status: Approved direction (phased plan below). Open questions at the end.

## Why

The user's favorite cluster-visualization tool, kube-ops-view (Henning Jacobs /
hjacobs), is dormant: GitHub repo archived Oct 2020 (project moved to Codeberg),
last release v23.5.0 in May 2023. The user wants it back, working on current
Kubernetes versions — and has decided to **detach from the upstream repo and run
this as a separate project**: a **fresh start** that keeps the code (and our
design/agent docs) but drops all upstream git history and tags, with a brand-new
initial commit pushed to a new repo under the user's own GitHub account.

### User profile (drives every decision)

- **Not a programmer.** All code work happens in agent sessions. Success is
  measured by "I can run it and see my clusters", never by code elegance.
- Uses **multiple clusters**, authenticating via **kubeconfig / kubectl proxy**.
- Wants the **UI modernized eventually** (separate later phase, not now).
- Must never need to run build/release commands by hand → **fully automated
  releases** are a first-class requirement.

## Findings (verified 2026-11-18, from the code at commit 87ed169)

### What is healthy (less work than feared)

- **The Kubernetes-facing code still targets current APIs**: core v1 Node/Pod
  queries via pykube, and usage metrics via `metrics.k8s.io/v1beta1`
  (metrics-server) in `kube_ops_view/kubernetes.py`. No Heapster anywhere in
  code — the README's Heapster mention is stale, not a code defect.
- Backend data flow is sound and compact (~1,500 LOC): discoverers
  (`cluster_discovery.py`: static URLs / cluster registry / kubeconfig / mock)
  → `update.py` polling loop with backoff → `stores.py` (MemoryStore, or
  RedisStore with pub/sub + redlock for multi-replica) → SSE at `/events`
  (json_delta diffs).
- Tests: 1 offline unit test (`tests/test_mock.py`), passes; runs without a
  cluster. Test gate is `make test` = pre-commit (all files) then pytest under
  coverage.

### What is broken / rotten

- **Docker build is broken**: `poetry.lock` is legacy format
  (`lock-version = "1.1"`) and Dockerfile runs `poetry install --no-dev`, a flag
  removed in Poetry 2.x (it installs unpinned Poetry at build time).
- **CI is dead**: only `.travis.yml`, which is stale (Python 3.7) and Travis OSS
  is shut down. No GitHub Actions.
- **Frontend toolchain is EOL-everywhere**: pixi.js 4, webpack 4, babel 6,
  eslint 4, node 14. Builds output to gitignored
  `kube_ops_view/static/build/`; the Flask `/` route 503s ("JavaScript
  application bundle not found") until a bundle exists. webpack 4 on modern
  Node needs `NODE_OPTIONS=--openssl-legacy-provider` (MD4 hashing).
- **Unmaintained dependencies**: pykube-ng (dormant on Codeberg; a `new-pykube`
  fork exists; `kr8s` is the modern alternative), redlock-py (archived),
  stups-tokens (Zalando-era; used by the non-OAuth token-file path and imported
  at module import time in `cluster_discovery.py`), json-delta (old but pure
  Python and works).
- OAuth login (flask-dance) is unused by this user but part of the code — keep
  compiling, do not prioritize.

### Constraints to respect

- **License**: the code is GPLv3. The fork must remain GPLv3 with original
  copyright/attribution (Henning Jacobs and contributors) preserved.
- `gevent.monkey.patch_all()` must stay the first import in
  `kube_ops_view/main.py`.
- **SSE contract**: the frontend parses json_delta patches client-side
  (`app/src/vendor/json_delta.js`). Any backend change must keep emitting
  compatible patch arrays, or the frontend delta layer must change with it.
  This is the subtlest part of the system.
- Versioning is CalVer (`v23.5.0`); runtime `__version__` is sed-replaced in
  `kube_ops_view/__init__.py` at Docker build from `git describe`. The pyproject
  `version` field is stale and unused.
- Local dev environment: poetry is not installed on the user's machine; system
  Python is 3.9.6 — run tests in a venv with the deps installed directly if
  poetry-based flows are unavailable.

## Decisions made

1. **Phased revival, minimal first**: get it working on current Kubernetes
   before any modernization. UI modernization is a later, separate phase.
2. **No Go rewrite** (considered and rejected): the pixi.js frontend — the soul
   of the tool — stays JavaScript regardless, so a rewrite only ports the thin
   data pump at high regression risk (json_delta contract, edge-case behavior)
   while delaying the user's actual goal. Agent sessions maintain the Python
   fine; packaging pain is neutralized by automated releases. Revisit only if
   dependency rot ever blocks real fixes.
3. **Fresh-start fork (decided 2026-11-18, supersedes "fork with history")**:
   delete `.git`, `git init` fresh, single new initial commit containing the
   code + `AGENTS.md` + this design doc + `LICENSE` (GPLv3, attribution to
   Henning Jacobs and contributors preserved); drop the Codeberg remote, all
   tags, and dead `.travis.yml`; push to a new GitHub repo under the user's
   account. A fresh-history snapshot of GPLv3 code is compliant as long as
   license and attribution stay intact.
4. **Automation-first**: tag → CI → lint/test/e2e → GHCR image + GitHub Release
   with notes. The user's upgrade path is "bump the image tag".

## The plan

### Phase 0 — Fresh start as a separate project
- Locally: drop `.git` and upstream remote, `git init` fresh, initial commit
  with the code + design/agent docs + LICENSE; remove `.travis.yml`.
- Rename the local project directory to the new project name (git is
  unaffected by directory renames).
- Create the new GitHub repo (user action in the browser, or via `gh` CLI if
  authenticated) and push the fresh history as `origin`.
- Light rebrand: new name in README (crediting upstream and linking it), image
  references in `deploy/`; keep CLI/import paths (`kube_ops_view`) stable for
  now to minimize churn — internal renames can ride along with Phase 4 polish.

### Phase 1 — Fix the build
- Migrate `pyproject.toml` to Poetry 2.x format; regenerate lock; Dockerfile
  `--no-dev` → `--only main`.
- Bump Python deps to current (Flask/gevent/click/pykube-ng stay — architecture
  unchanged); keep Python 3.11+ (evaluate 3.12 for the Docker image).
- `make appjs`: bump dockerized node to a current LTS +
  `--openssl-legacy-provider` for webpack 4; do not touch pixi/webpack versions
  yet.
- Replace Travis with GitHub Actions: lint (pre-commit) + pytest + docker build
  on PR/push.

### Phase 2 — Prove latest-Kubernetes support
- Spin up **two kind clusters** + metrics-server (mirrors the user's
  multi-cluster usage).
- Run the app with `--kubeconfig-path` + both contexts; verify: nodes/pods
  render, CPU/memory usage bars work (metrics.k8s.io), SSE live-updates (scale
  a deployment, watch the change), plus a quick `kubectl proxy` path check.
- Fix whatever surfaces (RBAC in `deploy/`, deprecations, content types).

### Phase 3 — Release automation
- On tag: CI runs lint → tests → kind e2e → builds image → pushes to GHCR →
  creates GitHub Release with generated notes.

### Phase 4 — User-facing polish
- Rewrite README for users (install via `kubectl apply -k deploy/` or
  `docker run`, config env vars, metrics-server prerequisite); repair
  `deploy/` kustomize manifests; refresh `docs/`.

### Later, separate effort — UI modernization
- pixi.js 8, bundler modernization (vite or webpack 5), possibly kr8s migration
  and auth rework. Planned only after the user has lived with the revived tool.

## Success criteria

- `make test` green in CI; e2e kind check green.
- UI renders two kind clusters including usage bars and live SSE updates.
- Tagging produces a published GHCR image + GitHub Release with zero manual
  steps.
- The user can point it at their real clusters with only a kubeconfig.

## Open questions (answer when ready; none block Phases 1–2)

1. ~~New project name~~ **Resolved 2026-11-18: `k8s-ops-view`**.
2. ~~Repo location~~ **Resolved: github.com/sharath94z/k8s-ops-view (public);
   images at `ghcr.io/sharath94z/k8s-ops-view`.**
3. Whether the revived tool should also be installable outside Docker (e.g.,
   published binaries) — nice-to-have, not planned.
