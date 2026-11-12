# Galdur-Software/.github

Shared GitHub Actions workflows for Galdur Software's products and client installs.
Nothing here holds secrets, host names or client data: a repo calls a workflow, and anything private
(secrets, runners, deploy targets) stays in that repo.

| Workflow | What it does |
|---|---|
| `node-ci.yml` | `actions/setup-node` with the npm cache, `npm ci`, then the repo's build command (default `npm run build`) and optional test command |
| `python-ci.yml` | uv virtualenv, `pip install -r requirements.txt`, then the repo's test command (default `python tests/smoke.py --strict`) |
| `security.yml` | gitleaks over the whole git history, zizmor over the repo's own workflows; plain binaries, checksum-verified |
| `docker-image.yml` | `docker build`; with `push: true` from a `vX.Y.Z` tag, push `ghcr.io/<owner>/<repo>:X.Y.Z` (must match `VERSION`) and `:sha-…`. Needs `packages: write` |

## Using one

```yaml
# .github/workflows/ci.yml in the calling repo
name: ci
on:
  push:
    branches: [main]
  pull_request:
permissions: {}
jobs:
  test:
    uses: Galdur-Software/.github/.github/workflows/python-ci.yml@<sha> # v1
    permissions:
      contents: read
  security:
    uses: Galdur-Software/.github/.github/workflows/security.yml@<sha> # v1
    permissions:
      contents: read
```

Pin the full commit SHA with the tag as a comment; Dependabot in the calling repo moves the pin when a new tag lands.

## Rules for workflows here

- Every third-party action pinned to a full commit SHA with its version as a comment. Dependabot keeps them current.
- `permissions: {}` at the top, then the least each job needs.
- `actions/checkout` with `persist-credentials: false`.
- Inputs reach shell steps through `env:`, never `${{ }}` inside `run:`.
- Downloaded tools are checksum-verified.
- `ci.yml` runs actionlint and zizmor on this repo itself.

## Releasing

Change, push, wait for `ci` to pass, then `git tag -a v1.x.y -m ... && git push --tags` and move `v1` along.
Breaking input changes get `v2`.
