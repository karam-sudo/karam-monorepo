# Submodules: create, link, and push

The parent repo `karam-monorepo` does not contain package source. Each package is its own GitHub repo. The parent stores only the URL and one commit SHA for that package.

Two pushes are always required:

1. Push the commit in the package repo.
2. Push the parent commit that records that package SHA.

Pushing the package alone leaves `main` on the old SHA. Pushing the parent alone does not upload package files.

Local parent:

```text
/home/karam/Documents/projects/karazoun/mkarazon/k8s-new/karam-monorepo
```

| Repo | Path in the parent | Remote |
|---|---|---|
| Parent | repo root | `https://github.com/karam-sudo/karam-monorepo.git` |
| api-utils | `packages/api-utils` | `https://github.com/karam-sudo/karam-api-utils.git` |
| plugin-alpha | `packages/plugin-alpha` | `https://github.com/karam-sudo/karam-plugin-alpha.git` |

## 1. Create the package repo

Do this before `git submodule add`. The remote needs at least one commit.

```bash
mkdir -p ~/src/plugin-beta
cd ~/src/plugin-beta
git init -b main

echo "# plugin-beta" > README.md
git add README.md
git commit -m "feat: initial plugin-beta"

git remote add origin git@github.com:karam-sudo/plugin-beta.git
git push -u origin main
```

Create the empty GitHub repo `karam-sudo/plugin-beta` before that push. `https://github.com/karam-sudo/plugin-beta.git` is the URL you will link. Use the SSH remote only on this machine for the push.

Copy `.github/workflows/ci.yaml` from `plugin-alpha` into the new package when it should run the parent quality and Trivy workflows. Set `plugin_name` to the folder name you will use under `packages/`. The sample file is `.github/samples/plugin-ci.yaml`.

## 2. Link it in the parent

From the parent, on `main`:

```bash
cd /home/karam/Documents/projects/karazoun/mkarazon/k8s-new/karam-monorepo
git checkout main
git pull origin main

git submodule add https://github.com/karam-sudo/plugin-beta.git packages/plugin-beta
```

That command clones the package into `packages/plugin-beta` and stages two things:

- `.gitmodules` gains a block like this:

```ini
[submodule "packages/plugin-beta"]
	path = packages/plugin-beta
	url = https://github.com/karam-sudo/plugin-beta.git
```

- `packages/plugin-beta` is a gitlink. `git ls-tree HEAD packages/plugin-beta` shows mode `160000` and one SHA. That SHA is the package commit `main` was on when you added it.

If `packages/plugin-beta` already exists as an empty directory, remove that empty directory, then run `git submodule add` again.

## 3. Commit the link and push the parent

```bash
git add .gitmodules packages/plugin-beta
git commit -m "feat(packages): add plugin-beta"
git push origin main
```

After this push, GitHub's parent repo shows `packages/plugin-beta` as a link to the package SHA, not as copied files.

Check the link:

```bash
git submodule status
git ls-tree HEAD packages
```

A line that starts with `-` means the folder is recorded but the files are not checked out.

## 4. Change the package later

A submodule checkout is detached. Check out `main` before editing.

```bash
cd /home/karam/Documents/projects/karazoun/mkarazon/k8s-new/karam-monorepo/packages/plugin-beta
git checkout main
git pull origin main

# edit files
git add .
git commit -m "fix: describe the change"
git push origin main
```

That push updates `karam-sudo/plugin-beta` only. Return to the parent and record the new SHA:

```bash
cd /home/karam/Documents/projects/karazoun/mkarazon/k8s-new/karam-monorepo
git add packages/plugin-beta
git commit -m "chore(packages): bump plugin-beta"
git push origin main
```

`git status` in the parent shows `packages/plugin-beta` as modified when the checked-out SHA differs from the SHA `main` records. `git diff packages/plugin-beta` prints the old SHA and the new SHA.

## 5. Clone the parent with the packages

```bash
git clone --recurse-submodules https://github.com/karam-sudo/karam-monorepo.git
cd karam-monorepo
```

If the clone is already there and `packages/plugin-beta` is empty:

```bash
git checkout main
git pull origin main
git submodule update --init --recursive
```

`git submodule update --init` checks out the SHAs stored by the parent. It does not move a package to the latest commit on its own `main`.

To move one package to the tip of its `main` and then record that in the parent:

```bash
cd packages/plugin-beta
git checkout main
git pull origin main
cd ../..
git add packages/plugin-beta
git commit -m "chore(packages): bump plugin-beta"
git push origin main
```

## 6. What each push triggers

| Push | Repo that changes | What runs |
|---|---|---|
| Package `main` | `karam-sudo/plugin-beta` | Quality, CodeQL, Trivy, then GHCR push |
| Parent `main` | `karam-sudo/karam-monorepo` | Parent CodeQL and the submodule pointer check |

The package workflow file calls:

```text
karam-sudo/karam-monorepo/.github/workflows/quality-plugin.yaml@main
karam-sudo/karam-monorepo/.github/workflows/build-api-plugin.yaml@main
```

Those files must already exist on the parent's `main` before the package push. Push the parent workflows first, then the package.

## Order for a new package

1. Create the GitHub repo and push its first commit to `main`.
2. Add `.github/workflows/ci.yaml` in that repo and push it.
3. In the parent, `git submodule add <https-url> packages/<name>`.
4. Commit `.gitmodules` and `packages/<name>` in the parent.
5. Push the parent `main`.
