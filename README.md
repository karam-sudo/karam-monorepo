# karam-monorepo

Training copy of the octobots layout: one parent repo, package repos attached as submodules, and the security and build workflows in the parent.

This is not the production platform. It does not call Karzoun registries, Slack, or private repositories. Images go to GHCR under `karam-sudo`.

## Repos

| Repo | Role |
|---|---|
| [karam-monorepo](https://github.com/karam-sudo/karam-monorepo) | Parent. Branch `main` stores one commit SHA per package. |
| [karam-api-utils](https://github.com/karam-sudo/karam-api-utils) | Shared package submodule. |
| [karam-plugin-alpha](https://github.com/karam-sudo/karam-plugin-alpha) | Sample plugin. Its CI calls the parent workflows. |

```
karam-monorepo/
├── .github/workflows/     # CodeQL, Trivy, reusable quality and build
├── packages/
│   ├── tsconfig.api.jsonc # shared TypeScript config, lives in the parent
│   ├── api-utils/         # gitlink
│   └── plugin-alpha/      # gitlink
└── scripts/health.js      # small JS file so parent CodeQL has code to scan
```

Full create, link, and push steps: [SUBMODULES.md](SUBMODULES.md).

What each workflow does: [WORKFLOWS.md](WORKFLOWS.md).

## Match what `main` records

```bash
git clone https://github.com/karam-sudo/karam-monorepo.git
cd karam-monorepo
git checkout main
git pull origin main
git submodule update --init --recursive
```

`git submodule update --init` checks out the SHAs recorded by the parent. `git submodule update --remote` moves a package to its own branch tip and is a different action: after that you commit the new SHA in this repo.

## Change a package

```bash
cd packages/plugin-alpha
git checkout main
# edit, commit, push to karam-plugin-alpha

cd ../..
git add packages/plugin-alpha
git commit -m "chore(packages): bump plugin-alpha"
git push origin main
```

Pushing the package starts its CI: type-check, Prettier, CodeQL, then image build, Trivy, and a GHCR push. Pushing the parent only moves the pointer.

## Secops

Workflow map: [.github/workflows/README.md](.github/workflows/README.md).

- Parent CodeQL runs on every push to `main`. Results land on the Security tab of `karam-monorepo`.
- Plugin CodeQL and Trivy run from `karam-plugin-alpha` and land on that repo's Security tab.
- Trivy fails the build on fixed HIGH or CRITICAL findings. The sample image is `FROM scratch`, so the first run is clean.
- To watch the gate block a push, change `packages/plugin-alpha/Dockerfile` to `FROM node:22-bookworm-slim` in the plugin repo and push `main`.

Repos are public so code scanning and SARIF upload work on a personal GitHub account.
