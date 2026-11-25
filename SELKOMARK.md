# Selkomark Bevy Distribution

This repository is Selkomark's fork of [bevyengine/bevy](https://github.com/bevyengine/bevy).
It tracks upstream Bevy releases and adds Selkomark-specific changes and crates on top.

Read this document before making your first commit.

## Remotes

| Remote     | URL                                       | Purpose                                     |
|------------|-------------------------------------------|---------------------------------------------|
| `origin`   | `git@github.com:Selkomark/bevy.git`       | The Selkomark fork. All pushes go here.     |
| `upstream` | `https://github.com/bevyengine/bevy.git`  | Official Bevy. Fetch only, never push.      |

Set up a fresh clone:

```sh
git clone git@github.com:Selkomark/bevy.git
cd bevy
git remote add upstream https://github.com/bevyengine/bevy.git
git remote set-url --push upstream DISABLED   # prevents accidental pushes to Bevy
git fetch upstream --tags
git switch master

# Recommended: remember conflict resolutions across upstream merges
git config rerere.enabled true
git config merge.conflictstyle zdiff3

# Required: without this, `gh pr create` targets bevyengine/bevy instead of the fork
gh repo set-default Selkomark/bevy
```

## Branches

| Branch            | Contents                                         | Who commits                         |
|-------------------|--------------------------------------------------|-------------------------------------|
| `master`          | The Selkomark distro: latest Bevy release plus our changes. Default branch. | Merges from feature branches and upstream releases only. |
| `main`            | Exact mirror of `upstream/main`.                  | Nobody. Fast-forward from upstream only. |
| `feat/<name>`     | New Selkomark features.                           | Developers, branched from `master`. |
| `fix/<name>`      | Bug fixes to the distro.                          | Developers, branched from `master`. |
| `upgrade/<tag>`   | Integration branch for merging a new Bevy release.| Maintainer performing the upgrade.  |

Rules:

- `master` and `main` are different branches. `master` is ours; `main` is Bevy's. Do not mix them up.
- Never commit directly to `main`. It must stay identical to `upstream/main`.
- Never commit directly to `master`. All changes land via pull request.
- Always branch from `master`, and open pull requests against `master`.
- Do not open pull requests against `bevyengine/bevy` from this repository's branches unless
  the change is intentionally being contributed upstream (see [Contributing upstream](#contributing-upstream)).

## Day-to-day workflow

```sh
git switch master
git pull
git switch -c feat/my-feature
# ... work, commit ...
git push -u origin feat/my-feature
gh pr create --base master
```

## Upgrading to a new Bevy release

New Bevy releases are merged into `master` through an `upgrade/<tag>` branch and a pull request.
The full procedure, including conflict resolution, verification, and rollback, is in
[SELKOMARK_UPSTREAM_MERGE.md](SELKOMARK_UPSTREAM_MERGE.md).

### Keeping `main` in sync (optional)

```sh
git fetch upstream
git switch main
git merge --ff-only upstream/main
git push origin main
```

If `--ff-only` fails, someone committed to `main`. Reset it: `git reset --hard upstream/main`.

## Where to put Selkomark code

Upstream merges are cheap only if we touch as few upstream files as possible.

1. **Prefer new crates.** Put new functionality in `crates/selkomark_<name>/` and add it to the
   workspace `members` in the root `Cargo.toml`. Upstream never touches these, so they never conflict.
2. **Modify upstream files only when unavoidable.** Keep the change minimal and mark it:

   ```rust
   // SELKOMARK: <short reason>
   ```

   For multi-line changes, wrap them:

   ```rust
   // SELKOMARK-BEGIN: <short reason>
   ...
   // SELKOMARK-END
   ```

   Find every modification with `git grep -n SELKOMARK`.
3. **Do not reformat or reorganize upstream code.** Unrelated diffs become conflicts on every upgrade.

## Versioning

Distro releases are tagged `selkomark-v<bevy version>-<n>`, where `<n>` starts at 1 and increments
for each distro release on the same Bevy version:

```sh
git switch master
git pull
git tag -a selkomark-v0.20.1-1 -m "Selkomark distro on Bevy 0.20.1"
git push origin selkomark-v0.20.1-1
```

Never create tags that start with `v`; that namespace belongs to upstream.

## Contributing upstream

If a change is generally useful, contribute it to Bevy so we stop carrying it:

```sh
git fetch upstream
git switch -c upstream/<name> upstream/main
# apply the change, following CONTRIBUTING.md
git push -u origin upstream/<name>
gh pr create --repo bevyengine/bevy --base main
```

Once upstream merges it and it ships in a release, remove our copy during the next upgrade.

## Quick reference

| Task                          | Command                                              |
|-------------------------------|------------------------------------------------------|
| Start a feature               | `git switch master && git pull && git switch -c feat/x` |
| See all Selkomark patches     | `git grep -n SELKOMARK`                              |
| Diff distro vs. its Bevy base | `git diff $(git describe --tags --match 'v*' --abbrev=0) --stat` |
| List upstream releases        | `git tag -l 'v*' --sort=-v:refname \| head`          |
| List distro releases          | `git tag -l 'selkomark-v*' --sort=-v:refname`        |
