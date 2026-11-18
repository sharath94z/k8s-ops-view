# AGENTS.md

k8s-ops-view: read-only WebGL dashboard for multiple K8s clusters — a
fresh-start revival of hjacobs/kube-ops-view (upstream dormant since 2023,
GitHub mirror archived 2020: https://codeberg.org/hjacobs/kube-ops-view). GPLv3;
keep upstream attribution.
Two apps wired together at build time: Python backend (`kube_ops_view/`, Flask + gevent, SSE) and JS frontend (`app/`, pixi.js + webpack).

**Active project: revival as a separate fork** — read
`docs/superpowers/specs/2026-11-18-revival-design.md` before doing any work
here. It holds the verified findings, decisions (phased plan, no rewrite, GPLv3
constraint, automation-first releases), and the user's profile (not a
programmer — deliver runnable results and automated releases, not instructions).

## Commands

- Setup: `make install` (runs `poetry install`). Python >= 3.11 required (`pyproject.toml`, Dockerfile).
- Full gate: `make test` = `poetry install` → `pre-commit run --all-files` (black, mypy, flake8, pydocstyle, yamllint, reorder-python-imports) → pytest under coverage. Lint failures abort before tests run.
- Single test: `poetry run pytest tests -v`. Tests are offline unit tests, no cluster needed.
- Autofix formatting/lint: `poetry run pre-commit run --all-files` (hooks rewrite files: black, reorder-python-imports, end-of-file-fixer).
- Frontend build: `make appjs` (runs `node:14.0-slim` in Docker) or `cd app && npm install && npm run build`; watch mode: `(cd app && npm start &)`.
- Run app without a cluster: `python3 -m kube_ops_view --mock --debug`, then open http://localhost:8080.

## Gotchas

- The UI returns 503 ("JavaScript application bundle not found") until `kube_ops_view/static/build/app*.js` exists — webpack's output dir, gitignored. Build the frontend before expecting `/` to load.
- `gevent.monkey.patch_all()` must stay the first import in `kube_ops_view/main.py`, before all other imports.
- Dockerfile installs unpinned Poetry and calls `poetry install --no-dev`; Poetry 2.x removed `--no-dev`, and `poetry.lock` is legacy format (`lock-version = "1.1"`). Image builds need Poetry < 2 or both the flag and lock format updated together.
- Pre-commit hooks marked `stages: [push]` (bandit, safety, pyupgrade, `poetry check`) are skipped by `make test`'s `pre-commit run --all-files`; include them via `pre-commit run --hook-stage push --all-files`.
- Commit messages are linted by gitlint (commit-msg stage).
- Test files must be named `test_*.py` (name-tests-test hook).
- flake8 allows long lines (`max-line-length=240`); don't hand-reflow them.
- Versioning: CalVer git tags (`v23.5.0`). Runtime `__version__` is sed-replaced in `kube_ops_view/__init__.py` at Docker build from `git describe`; the `version` field in pyproject.toml is stale and unused. `make version` updates the image tag in `deploy/*.yaml`.

## Architecture

Data flow (backend): discoverers in `cluster_discovery.py` (static `CLUSTERS` URLs / cluster registry / kubeconfig / mock) → `update.py` polling loop (`update_clusters`) → per-cluster query in `kubernetes.py` → `stores.py` (MemoryStore, or RedisStore for pub/sub + redlock across replicas) → SSE at `/events` (json_delta diffs sent to the browser).

- `kube_ops_view/main.py` — click CLI (`python3 -m kube_ops_view`, also the Docker ENTRYPOINT), Flask routes, gevent pywsgi server, background update loop.
- `app/` — frontend source in `app/src/` (pixi.js WebGL rendering, ES6, webpack 4 + babel), builds into `kube_ops_view/static/build/` and is served by `kube_ops_view/templates/index.html`. `npm run lint` = eslint.
- `tests/` — pytest.
- `deploy/` — kustomize manifests (`kubectl apply -k deploy`); `openshift/` — OpenShift example deployments; `docs/` — Sphinx RST published on readthedocs.
