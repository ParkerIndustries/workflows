# workflows

Reusable GitHub Actions workflows shared across Parker Industries / MajorDom Systems repos.
Single source of truth for CI/CD logic — consuming repos hold only a thin stub that
triggers on real events (`push`, `pull_request`, `workflow_dispatch`) and calls into one
of these via `uses:`. See [python-template](https://github.com/ParkerIndustries/python-template)
for ready-to-copy stubs.

## Available workflows

### `test.yml` — QA & Tests

Runs `poe check --ci` (ruff, type-check, pytest, `poetry build`/`check`, plus a
`git diff --exit-code` guard) via Poetry, across a Python-version matrix.

| Input | Default | Description |
|---|---|---|
| `python-versions` | `'["3.12", "3.13"]'` | JSON array of Python versions for the QA matrix. Bump here to migrate every consumer in sync; override per-repo for a custom range |
| `working-directory` | `"."` | Subdirectory containing `pyproject.toml`, for monorepos |
| `setup-script` | `""` | Shell run after `poetry install`, before `poe check` (per matrix leg). For system packages, Cython `build_ext`, starting service containers, etc. |
| `post-run-script` | `""` | Shell run after `poe check`, **always** (even on failure). For teardown/cleanup |

A calling job that uses a reusable workflow **cannot have its own `steps`**, and a separate
job runs on a fresh runner — so environment setup the checks depend on has to go through
`setup-script`, which runs on the same runner.

```yaml
jobs:
  test:
    uses: ParkerIndustries/workflows/.github/workflows/test.yml@v1
    # with:
    #   python-versions: '["3.12", "3.13", "3.14"]'
    #   setup-script: |
    #     sudo apt-get update && sudo apt-get install -y espeak-ng
    #     poetry run python setup.py build_ext --inplace
    #   post-run-script: docker compose down
```

The matrix runs under a `qa` job (one leg per version) plus a stable `check` gate job that
passes only if every leg passed. **Branch protection should require the `check` gate**
(e.g. `test / check`, prefixed by your calling job's id) — not the individual `py3.x` legs,
which change with the matrix.

### `release.yml` — QA → version bump → PyPI publish → tag → GitHub Release

Runs `test.yml` (QA matrix) and a `pip-audit` dependency-vulnerability gate first — a
vulnerable dep blocks the release, though it deliberately does **not** gate everyday
PRs/pushes (pip-audit isn't part of `poe check`). On success: bumps the patch version on
the dev branch, builds, publishes to PyPI via
[trusted publishing](https://docs.pypi.org/trusted-publishers/) (no `PYPI_API_TOKEN`
needed — configure a trusted publisher on PyPI first), fast-forwards the dev branch into
the stable branch, tags the release, and creates a signed GitHub Release with an
auto-generated changelog.

Build is single (one `py3-none-any` wheel + sdist) by default. Set `matrix-build: true`
for compiled/Cython projects to build a wheel per version in `python-versions` (plus one
sdist); leave it off for pure Python, where a matrix build would produce colliding
identical wheels.

The changelog lists commit subjects since the last tag under a **Changes** heading. If
`docs-url` is set, it additionally lists any `docs/**.md` pages added or modified since the
last release under **New Documentation** / **Updated Documentation** headings, as links to
the hosted docs site (index pages map to their directory root to match mkdocs'
`use_directory_urls`). Repos without published docs leave `docs-url` unset and get the
commits-only changelog.

| Input | Default | Description |
|---|---|---|
| `python-version` | `"3.12"` | Single version for bump/audit, and the build unless `matrix-build` |
| `python-versions` | `'["3.12", "3.13"]'` | JSON array forwarded to the QA matrix (and the build if `matrix-build`) |
| `matrix-build` | `false` | Build a wheel per version (compiled/Cython projects). `python-version` must be one of `python-versions` (it builds the sdist) — validated up front, fails red otherwise |
| `working-directory` | `"."` | Subdirectory containing `pyproject.toml` |
| `setup-script` | `""` | Runs before QA checks and before the build (compiled extensions need `build_ext` here too) — see `test.yml` |
| `post-run-script` | `""` | Teardown after the QA checks, always — see `test.yml` |
| `pypi-package-name` | *(required)* | PyPI distribution name, for the release environment URL |
| `stable-branch` | `"main"` | Protected release branch — set to `master` if that's your convention |
| `dev-branch` | `"develop"` | Integration branch releases are cut from |
| `docs-url` | `""` | Hosted docs base URL (e.g. `https://stark.markparker.me`); enables the docs sections in the changelog. Empty = commits-only |

```yaml
jobs:
  release:
    uses: ParkerIndustries/workflows/.github/workflows/release.yml@v1
    with:
      pypi-package-name: your-pypi-name
```

Requires the calling repo to have a `release` GitHub Environment (deployment branch
`develop` only, required reviewer) and PyPI trusted publishing configured — see
python-template's README for the one-time setup steps.

### `dependabot-automerge.yml` — Auto-merge Dependabot PRs

Two jobs, selected by the caller's trigger:

- `automerge-new` — on a `pull_request` event from Dependabot, enables auto-merge (squash)
  on the single PR that triggered it; GitHub merges once its required checks pass.
- `automerge-all-open` — on `workflow_dispatch`, enables auto-merge on every currently-open
  Dependabot PR targeting `develop` at once (manual backfill).

```yaml
permissions:
  contents: write
  pull-requests: write
jobs:
  automerge:
    uses: ParkerIndustries/workflows/.github/workflows/dependabot-automerge.yml@v1
```

> The `permissions:` block **must** be set here in the calling repo, not just in this
> reusable workflow: Dependabot-triggered `pull_request` runs get a read-only `GITHUB_TOKEN`
> by default, and a reusable workflow can only narrow permissions, never elevate them. The
> repo also needs **Settings → General → Allow auto-merge** enabled.

### `docs-publish.yml` — Publish mkdocs-material docs to GitHub Pages

Builds and deploys docs via `mkdocs gh-deploy --force`. Caches mkdocs-material's build
cache by ISO week. Optional — only relevant if the repo ships `mkdocs.yml` + a `docs/`
tree (see python-template's `mkdocs.yml` starter, which includes `mkdocs-llmstxt` for an
`llms.txt`/`llms-full.txt` export alongside the rendered site).

| Input | Default | Description |
|---|---|---|
| `python-version` | `"3.12"` | Python version to install |
| `working-directory` | `"."` | Subdirectory containing `mkdocs.yml` |

```yaml
jobs:
  deploy:
    uses: ParkerIndustries/workflows/.github/workflows/docs-publish.yml@v1
```

Requires **Settings → Pages → Source: Deploy from a branch → `gh-pages`** enabled once,
and the calling job needs `permissions: contents: write` (already set inside this
reusable workflow's own job — no action needed in the caller beyond triggering it).

## Versioning

Consumers pin a **moving major tag** — `@v1` (the actions/checkout convention). Backward-
compatible changes (new optional inputs, fixes) land on `main` and then **`v1` is moved
forward** to that commit, so every consumer picks them up on their next run with no edits.
**Breaking changes cut a new major** (`v2`); consumers stay on `@v1` until they opt in.

So: prefer additive changes; when you must break an interface, tag `v2` rather than moving
`v1`. Internal cross-references (e.g. `release.yml` calling `test.yml`) also pin `@v1`, so a
consumer on `release.yml@v1` gets `test.yml@v1` consistently — keep those refs on the same
major when moving the tag.

Moving the tag after a compatible change:

```sh
git tag -f v1 && git push origin v1 --force
```

Optionally also push an immutable `vX.Y.Z` tag for consumers who prefer exact pins +
Dependabot-driven bumps (each bump then runs through their own CI before auto-merging).

## License

Apache License 2.0 — see [LICENSE](LICENSE).
