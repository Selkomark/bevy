# Merging Bevy releases into `master`

This document is the procedure for bringing a new [bevyengine/bevy](https://github.com/bevyengine/bevy)
release into the Selkomark distro branch, `master`. For the general branch model, see
[SELKOMARK.md](SELKOMARK.md).

## Principles

- **Merge release tags, never `upstream/main`.** `master` is always based on a published Bevy
  release (`v0.21.0`, `v0.21.1`, ...). Upstream `main` is unreleased and changes daily.
- **Merge, never rebase or cherry-pick.** A merge keeps Bevy's commits unchanged in our history,
  so `git` knows exactly which upstream commits we already have, and the next upgrade only has to
  deal with what is new.
- **Every upgrade goes through an `upgrade/<tag>` branch and a pull request.** `master` must build
  and pass tests at all times.
- **Upgrade PRs are merged with a merge commit.** Never use "Squash and merge" or "Rebase and merge"
  on an upgrade PR. Squashing discards the link to Bevy's history; the next upgrade would then
  conflict on every file Bevy touched since the fork.

## Before you start

One-time setup per clone (skip if already done, see [SELKOMARK.md](SELKOMARK.md#remotes)):

```sh
git remote add upstream https://github.com/bevyengine/bevy.git
git remote set-url --push upstream DISABLED
git config rerere.enabled true
git config merge.conflictstyle zdiff3
gh repo set-default Selkomark/bevy     # otherwise `gh pr create` targets bevyengine/bevy
```

Check what is available and what we are on:

```sh
git fetch upstream --tags
git tag -l 'v*' --sort=-v:refname | head                 # latest upstream releases
git describe --tags --match 'v*' --abbrev=0 master        # Bevy release master is based on
```

Choose the target tag:

| Situation                                  | Target                                   |
|--------------------------------------------|------------------------------------------|
| Patch release (`0.20.1` → `0.20.2`)        | Merge the patch tag directly.            |
| Next minor release (`0.20.x` → `0.21.0`)   | Merge the `.0` tag, then later patches.  |
| Several minor releases behind              | Merge one minor release at a time (`0.21.0`, then `0.22.0`, ...). Each step has its own migration guide and smaller conflicts. |
| Release candidates (`v0.21.0-rc.1`)        | Do not merge into `master`. Try them on a throwaway branch only. |

Read the migration guide for the target version before merging:
<https://bevy.org/learn/migration-guides/>. Bevy minor releases contain breaking API changes that
will affect our crates even where git reports no conflict.

## Procedure

### 1. Create the upgrade branch

```sh
git switch master
git pull
git switch -c upgrade/v0.21.0
```

### 2. Merge the release tag

```sh
git merge --no-ff v0.21.0 -m "Merge Bevy v0.21.0"
```

If there are no conflicts, go to step 4.

### 3. Resolve conflicts

List conflicted files:

```sh
git status --short | grep '^UU\|^AA\|^DU\|^UD'
```

Conflicts will almost only occur in upstream files we modified. List all our modifications:

```sh
git grep -n 'SELKOMARK'
```

For each conflicted file:

1. Open it. With `zdiff3`, each conflict shows three parts: ours (`<<<<<<<`), the common
   ancestor (`|||||||`), and Bevy's new version (`>>>>>>>`). Compare ancestor to Bevy's version to see
   what Bevy changed, then re-apply our change on top of Bevy's version.
2. Keep the `// SELKOMARK:` marker on the re-applied change.
3. If Bevy now provides what our patch did, drop our patch and take Bevy's version:
   `git checkout --theirs <file>`.
4. `git add <file>`.

Common conflict locations:

| File                      | Cause                                          | Resolution |
|---------------------------|------------------------------------------------|------------|
| Root `Cargo.toml`         | We added `crates/selkomark_*` to the workspace; Bevy bumped versions or changed members. | Take Bevy's version, then re-add our workspace members. |
| `README.md`               | Our Selkomark banner at the top.               | Take Bevy's version, then re-add the banner line. |
| `crates/selkomark_*/Cargo.toml` | Not a conflict, but Bevy dependency versions must match the new release. | Update `bevy_*` dependency versions to the new release (see step 4). |

`rerere` records each resolution. If the same conflict appears in a later upgrade, git applies the
recorded resolution automatically; review it, then `git add` the file.

When all files are resolved:

```sh
git commit          # completes the merge commit
```

To abandon the merge at any point and start over:

```sh
git merge --abort
```

### 4. Update Selkomark crates for the new release

- Bump `bevy_*` dependency versions in every `crates/selkomark_*/Cargo.toml` to the new release.
- Fix compile errors caused by Bevy API changes, using the migration guide.
- Commit these as separate commits on the upgrade branch (not inside the merge commit), for example
  `Update selkomark_foo for Bevy 0.21`. This keeps the merge commit limited to the upstream merge.

### 5. Verify

```sh
cargo build --workspace
cargo test --workspace
cargo clippy --workspace --all-targets
cargo run --example breakout    # sanity check that rendering, input and audio work
```

Run any Selkomark examples or downstream projects that depend on this distro against the branch.

Confirm every Selkomark patch survived the merge:

```sh
git grep -n 'SELKOMARK'
```

Compare against the list on `master` (`git grep -n SELKOMARK master`). Each patch must still be
present, or be removed deliberately because Bevy made it unnecessary.

### 6. Open the pull request

```sh
git push -u origin upgrade/v0.21.0
gh pr create --base master --title "Upgrade to Bevy v0.21.0"
```

In the PR description, include:

- The previous and new Bevy versions.
- Link to the Bevy release notes and migration guide.
- Conflicts resolved, and how.
- Selkomark patches dropped because Bevy now covers them.
- Verification performed.

On GitHub, confirm the PR base is `Selkomark/bevy` → `master`. The web UI defaults forked repositories
to the upstream repository; change it if needed.

### 7. Merge and tag

Merge the PR with **"Create a merge commit"**. Then tag the distro release:

```sh
git switch master
git pull
git tag -a selkomark-v0.21.0-1 -m "Selkomark distro on Bevy 0.21.0"
git push origin selkomark-v0.21.0-1
git branch -d upgrade/v0.21.0
git push origin --delete upgrade/v0.21.0
```

### 8. Sync `main` (optional)

```sh
git switch main
git merge --ff-only upstream/main
git push origin main
```

## Rolling back

If a bad upgrade has already merged into `master`, revert the PR's merge commit instead of
resetting `master`:

```sh
git switch master
git pull
git switch -c revert/v0.21.0
git revert -m 1 <merge-commit-of-the-upgrade-PR>
git push -u origin revert/v0.21.0
gh pr create --base master --title "Revert Bevy v0.21.0 upgrade"
```

Important: after reverting a merge, git considers those upstream commits already merged. To retry
the upgrade later, first revert the revert commit (`git revert <revert-commit>`) on the new upgrade
branch, then fix the problems and continue from step 4.

Never force-push `master`.

## Checklist

- [ ] `git fetch upstream --tags`
- [ ] Read the migration guide for the target version
- [ ] `upgrade/<tag>` branch created from up-to-date `master`
- [ ] `git merge --no-ff <tag>` and conflicts resolved
- [ ] All `SELKOMARK` patches present or deliberately removed
- [ ] Selkomark crates updated to the new Bevy version
- [ ] `cargo build`, `cargo test`, `cargo clippy` pass on the workspace
- [ ] PR opened against `Selkomark/bevy` `master`
- [ ] Merged with a merge commit (not squash, not rebase)
- [ ] Distro tag `selkomark-v<version>-<n>` pushed
