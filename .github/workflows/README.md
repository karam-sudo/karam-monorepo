# Training workflows

Same shape as the octobots parent: reusable workflows live here, and each package repo calls them.

| Workflow | Runs on | What it checks |
|---|---|---|
| `codeql.yaml` | This repo, push and PR to `main` | CodeQL SAST of the parent |
| `submodule-pointers.yaml` | This repo | Every `.gitmodules` path is a gitlink (mode `160000`) |
| `commitlint.yaml` | Pull requests on this repo | Conventional Commit PR title |
| `label-pr.yaml` | Pull requests on this repo | Path and size labels |
| `stale.yaml` | Daily | Stale issues and PRs |
| `quality-plugin.yaml` | Called by a package repo | `tsc`, Prettier, CodeQL |
| `pr-title-plugin.yaml` | Called by a package repo | Conventional Commit PR title |
| `build-api-plugin.yaml` | Called by a package repo | Image build, Trivy, SARIF upload, GHCR push |

Package CI example: [../samples/plugin-ci.yaml](../samples/plugin-ci.yaml).

Trivy uses the same gate as production: fixed `HIGH` and `CRITICAL` fail the job, and the image is pushed only after that scan succeeds. The training image is `FROM scratch`, so the first scan is clean. Change the package `Dockerfile` to `node:22-bookworm-slim` when you want to see the gate stop the push.

These repos are public so CodeQL and Trivy SARIF uploads work on a personal account. A private repo needs GitHub code scanning enabled before the upload step can succeed.
