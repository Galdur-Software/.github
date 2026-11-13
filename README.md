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
| `wp-ci.yml` | WordPress themes and plugins: `php -l` on several PHP versions (deprecations fail), every `.json` file must parse (theme.json, block.json, Pods' `pods.json`), the repo's test command, then its package command; the zips are uploaded as an artifact |
| `wp-site-check.yml` | The repo's site check script (Playwright, Chromium) against a preview or live URL; markdown report as artifact and job summary. Optional `login` secret for a password-protected preview |
| `wp-handoff-docs.yml` | The repo's command that builds handoff PDFs (guide, login sheet), shrunk with Ghostscript, kept as an artifact. Gets no secrets, by design |
| `wp-deploy-sftp.yml` | Deploys the packages from the successful CI run for one commit over SFTP (password, no shell needed), then checks the versions over the WordPress REST API with an application password. Needs `actions: read`. Call it from a `workflow_dispatch` workflow: pressing Run is the approval |

### WordPress sites: the whole pipeline

A site repo runs `security.yml`, `wp-ci.yml`, `wp-site-check.yml` and `wp-handoff-docs.yml` on every push, and
`wp-deploy-sftp.yml` (then `wp-site-check.yml` with `live: true`) only when a person starts its deploy workflow.
Required reviewers on environments are not available for private repos on GitHub Free, Pro or Team, so the
manual start is the gate. Redeploying an older commit is the rollback, for as long as its package artifact is kept.

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
