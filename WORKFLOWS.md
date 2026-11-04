# Workflows

The parent repo holds the workflow files. A package repo does not copy the build or scan steps. Its `.github/workflows/ci.yaml` calls the parent files with `workflow_call`.

```text
push or pull request on karam-plugin-alpha
        │
        ▼
plugin-alpha/.github/workflows/ci.yaml
        │
        ├── quality ──► karam-monorepo/.github/workflows/quality-plugin.yaml@main
        │                 ├── tsc + Prettier
        │                 └── CodeQL on the package repo
        │
        ├── pr-title ─► pr-title-plugin.yaml@main          (pull requests only)
        │
        └── build ────► build-api-plugin.yaml@main         (push to main, after quality)
                          ├── docker build (no push yet)
                          ├── Trivy HIGH/CRITICAL
                          ├── upload SARIF to the package Security tab
                          └── docker push to ghcr.io
```

The job runs in the package repo. Results, logs, and Security tab entries show on `karam-plugin-alpha`, even though the YAML lives in `karam-monorepo`. The caller must pass `plugin_name`. The sample caller is `.github/samples/plugin-ci.yaml`.

Push the parent `main` before the first package push. GitHub loads `quality-plugin.yaml@main` and `build-api-plugin.yaml@main` from the parent.

## Workflows that run on the parent

These start from events on `karam-monorepo`. They do not build package images.

### `codeql.yaml` — CodeQL on the parent

Runs on push to `main`, pull request to `main`, every Monday 04:00 UTC, and when you click Run workflow.

It checks out the parent with submodules off, so it scans parent files such as `scripts/health.js`. It does not scan `packages/plugin-alpha` source, because that code lives in the package repo and is scanned by `quality-plugin.yaml`.

Steps: init CodeQL for `javascript-typescript` with the `security-and-quality` query suite, autobuild, analyze. Alerts go to the Security tab of `karam-monorepo`.

### `submodule-pointers.yaml` — gitlink check

Runs on push and pull request to `main`.

It reads every `path` in `.gitmodules` and checks `git ls-tree` for that path. The mode must be `160000`, which is a gitlink (one commit SHA), not a normal directory of files. The job fails if a submodule path is missing or was committed as regular files.

Checkout uses `submodules: false`. The test only needs the pointer stored in the parent commit.

### `commitlint.yaml` — parent PR title

Runs when a pull request on the parent is opened, edited, synchronized, or reopened.

It uses `amannn/action-semantic-pull-request` to require a Conventional Commit title: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, or `revert`. A scope is optional. `feat(packages): add plugin-beta` passes. `updated stuff` fails. The check is on the PR title, not on every commit message.

### `label-pr.yaml` — labels on parent PRs

Runs when a parent pull request is opened, synchronized, or reopened. Two jobs run in parallel.

- Path labels come from `actions/labeler` and `.github/labeler.yml`. A change under `.github/` gets `ci`. A change to `codeql.yaml`, `quality-plugin.yaml`, or `build-api-plugin.yaml` gets `secops`. `packages/plugin-*` gets `plugin`. Markdown gets `documentation`.
- Size labels come from `codelytv/pr-size-labeler`: `size/xs` through `size/xl`. `size/xl` posts a comment and does not fail the check.

### `stale.yaml` — close inactive issues and PRs

Runs every day at 06:00 UTC, and when you click Run workflow.

`actions/stale` marks an issue or PR `stale` after 30 days without activity and closes it 5 days later. Items labeled `pinned` or `security` are left open.

### `dependabot.yml` — action updates

This is not a workflow file. Dependabot opens pull requests on the parent every Monday at 06:00 Asia/Amman when a GitHub Action version in `.github/workflows/` can be updated. Those PRs then go through `commitlint.yaml` and `label-pr.yaml`.

## Workflows a package calls

Nothing in the parent `on:` block starts these. They start only when a package `ci.yaml` uses them.

### `quality-plugin.yaml` — type-check, format, CodeQL

Called on every package push and pull request.

Inputs:

| Input | Default | Meaning |
|---|---|---|
| `plugin_name` | required | Folder name, for example `plugin-alpha` |
| `node_version` | `22.x` | Node used for `tsc` and Prettier |
| `skip_codeql` | `false` | `true` skips the CodeQL job |

**Lint job.** It builds one working tree from three checkouts:

1. Parent `main`, sparse: `packages/tsconfig.api.jsonc` and `package.json`, placed at `monorepo/`.
2. The package repo that started the workflow, placed at `monorepo/packages/<plugin_name>/`.
3. `karam-sudo/karam-api-utils` at `main`, placed at `monorepo/packages/api-utils/`.

Then `npm ci`, `tsc --noEmit`, and `prettier --check` run inside the package directory. `tsc` extends `../tsconfig.api.jsonc`, which is the file from the parent. The package depends on `@karam/api-utils` through `file:../api-utils`, which is the checkout from step 3.

**CodeQL job.** It checks out the package repo only and runs the same `javascript-typescript` / `security-and-quality` suite. Alerts land on the package repo. Set `skip_codeql: true` in the caller to run lint without this job.

### `pr-title-plugin.yaml` — package PR title

Called only when `github.event_name == 'pull_request'`.

Same Conventional Commit title rule as the parent `commitlint.yaml`, without a fixed scope list. A package PR titled `fix: handle empty payload` passes.

### `build-api-plugin.yaml` — image, Trivy, GHCR

Called on a push to package `main`, or a manual run, and only after `quality` succeeds (`needs: quality`). A pull request does not build an image.

Inputs:

| Input | Default | Meaning |
|---|---|---|
| `plugin_name` | required | Folder name and image name |
| `node_version` | `22.x` | Reserved; the training Docker build does not compile in this job |
| `docker_registry` | `ghcr.io` | Only `ghcr.io` is accepted |
| `tag` | branch name | Image tag. A push to `main` uses tag `main` |

The image name is `ghcr.io/<owner>/<plugin_name>:<tag>`, lowercased. For this account that is `ghcr.io/karam-sudo/plugin-alpha:main`.

Steps, in order:

1. Checkout the parent, the calling package, and `karam-api-utils` under `monorepo/packages/`. Credentials are not written into `.git/config`.
2. `docker build` the package directory, which contains the `Dockerfile`. The build writes `image.tar` and does not push.
3. Trivy `v0.76.1` scans that archive for vulnerabilities and secrets. The gate is fixed `HIGH` and `CRITICAL` (`ignore-unfixed: true`, `exit-code: 1`). Unfixed findings do not fail the job.
4. The SARIF file is uploaded to the package repo Security tab even when the scan fails.
5. If the scan succeeded, the job logs in to GHCR with `GITHUB_TOKEN` and pushes the same archive that was scanned. A failed scan does not push.
6. `image.tar` is deleted.

The training `Dockerfile` is `FROM scratch`, so the first scan has no OS packages to flag. Change it to `FROM node:22-bookworm-slim` in the package repo and push `main` when you want the gate to fail and skip the push.

## What runs for one change

| You push | Workflows |
|---|---|
| Parent `main` | `codeql.yaml`, `submodule-pointers.yaml` |
| Parent pull request | `codeql.yaml`, `submodule-pointers.yaml`, `commitlint.yaml`, `label-pr.yaml` |
| Package `main` | `quality-plugin.yaml`, then `build-api-plugin.yaml` |
| Package pull request | `quality-plugin.yaml`, `pr-title-plugin.yaml` |

Parent CodeQL and package CodeQL are different scans. The parent scan does not replace the package scan.
