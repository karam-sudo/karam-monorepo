# Add another package

1. Create an empty GitHub repo, with one commit on `main`.
2. From this parent, on `main`:

```bash
git submodule add https://github.com/karam-sudo/<repo>.git packages/<name>
git add .gitmodules packages/<name>
git commit -m "feat(packages): add <name>"
git push origin main
```

3. Copy `plugin-ci.yaml` into that repo as `.github/workflows/ci.yaml`.
4. Set `plugin_name` to the folder name under `packages/`.
5. Push the package repo. Quality runs on every push and pull request. The image build runs on a push to `main`.
