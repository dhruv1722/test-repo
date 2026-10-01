# Permanent Developer-Branch Git Workflow

This repository uses permanent developer branches rather than a new feature branch for every task:

```text
origin/main        Shared source of truth
origin/dhruvesh    Dhruvesh's permanent working branch
origin/jayesh      Jayesh's permanent working branch
origin/anand       Anand's permanent working branch
```

`main` is the shared, reviewed version of the project. Developers push their own work to their own remote branch and create a pull request (PR) into `main`.

This document records the workflow tested in this repository. It deliberately does not use temporary backup branches.

## Important Git concepts

### Local `main`, remote `main`, and `origin/main`

| Name | Meaning |
| --- | --- |
| `origin/main` | A local remote-tracking reference. `git fetch origin` updates it from GitHub's remote `main`. |
| `main` | The local `main` branch. It moves only when it is explicitly updated, for example with `git pull --ff-only origin main`. |
| `dhruvesh` | The local permanent Dhruvesh branch. It contains Dhruvesh's commits. |
| `origin/dhruvesh` | GitHub's copy of the Dhruvesh branch. `git push origin dhruvesh` updates it. |

The flow is local after fetching:

```text
GitHub main
   ↓ git fetch origin
local origin/main
   ↓ git merge origin/main
currently checked-out developer branch
```

`git fetch origin` does not change the visible files, local `main`, or the current working branch. `git merge` merges into the branch currently checked out, so `git switch dhruvesh` is essential before running it.

### A branch stores commits, not unstaged edits

An unstaged or uncommitted edit only exists in the working folder. It is not safely stored in `dhruvesh`, `origin/dhruvesh`, or `main`.

Do not rely on switching branches while the working tree is dirty. Git may refuse an unsafe operation, may carry a non-overlapping edit between branches, or leave a situation that is difficult to reason about. In the practical test for this repository, an intentionally unstaged test line did not remain after switching branches. The safe team rule is therefore absolute:

> Never switch, pull, merge, or start a merge preview with uncommitted work that matters.

First commit that work on the same permanent developer branch. A local `WIP` commit is valid; pushing it to the developer branch makes it available on GitHub but does not put it in `main`.

## Required pre-check

Start any integration only when `git status` shows a clean working tree:

```text
On branch dhruvesh
Your branch is up to date with 'origin/dhruvesh'.

nothing to commit, working tree clean
```

If the working tree is dirty, preserve the work first:

```bash
git switch dhruvesh
git status
git add <only-the-files-that-belong-to-this-work>
git commit -m "WIP: describe the current work"
git push origin dhruvesh
```

Avoid `git add .` as a normal habit. It can stage unrelated changes. Stage named files or carefully reviewed groups of files instead.

## Updating a developer branch from `main`

There are two equivalent methods. Use one method consistently for a single update.

### Method A: merge the latest GitHub `main` directly

This is the shortest method and does not require local `main` to be current:

```bash
git status
git fetch origin
git switch dhruvesh
git merge --no-commit --no-ff origin/main
```

### Method B: update local `main`, then merge it

This is useful when the developer wants a local `main` checkout as a reference or wants to inspect/test it first:

```bash
git status
git switch main
git pull --ff-only origin main
git switch dhruvesh
git merge --no-commit --no-ff main
```

`git pull --ff-only origin main` fetches the remote branch and advances local `main` only when it can fast-forward safely. Do not use a plain `git pull` for this team procedure because its configured merge/rebase behavior may differ between machines.

After Method B, do not run another `git fetch origin` and then merge local `main` while assuming it is still the latest remote commit. If GitHub changed again, use `origin/main` directly or update local `main` again.

## What the merge preview does

```bash
git merge --no-commit --no-ff main
```

or:

```bash
git merge --no-commit --no-ff origin/main
```

This command does **not** create a merge commit yet. It does, however, apply the proposed merge result to the working tree and index (staging area). Therefore visible file contents can change during the preview.

It is safe to review because there are two deliberate choices:

```bash
# Reject the pending integration and restore the developer branch exactly
git merge --abort

# Accept the pending integration
git commit -m "Merge main into dhruvesh"
git push origin dhruvesh
```

The merge preview is not a way to keep two incompatible versions of one line. It is a way to inspect the combined result before creating a merge commit.

### Inspect the pending merge

```bash
git status
git diff --cached --name-only
git diff --cached --name-status
git diff --cached
```

Use `git diff --cached`, not plain `git diff`, because a clean merge stages its proposed changes automatically.

If Git reports a conflict, it will show an unmerged path such as:

```text
both modified: dhruvesh.txt
```

Git preserves both committed histories. Edit the file to the intended final content, then:

```bash
git add dhruvesh.txt
git commit
git push origin dhruvesh
```

If the decision cannot be made yet, use `git merge --abort`; do not use `ours`, `theirs`, a hard reset, or a force push merely to remove the conflict.

## Expected merge outcomes

| Situation | Git result |
| --- | --- |
| Only `main` changed a file/line since the common base | The `main` version appears in the developer branch's merge result. |
| The developer branch and `main` changed different files or different lines | Git normally keeps both changes automatically. |
| The developer branch and `main` changed the same lines differently | Git creates a merge conflict. A developer must choose the correct final content. |

If Dhruvesh has not committed a change to a line and `main` changed it, Git has no Dhruvesh change to preserve for that line. Taking the `main` version is the correct source-of-truth integration result.

## Scenario 1: uncommitted Dhruvesh work while another PR reaches `main`

### Situation

Umesh (or any other developer) pushed to their personal branch and their PR was merged into remote `main`. Dhruvesh has local edits that are unstaged and uncommitted.

### Correct workflow

1. Do not switch to `main` yet.
2. Commit Dhruvesh's work on the existing `dhruvesh` branch.
3. Push the commit to `origin/dhruvesh` if remote preservation is required.
4. Update `main` and preview the merge back into `dhruvesh`.

```bash
git switch dhruvesh
git status
git add dhruvesh.txt
git commit -m "WIP: Dhruvesh current work"
git push origin dhruvesh

git switch main
git pull --ff-only origin main

git switch dhruvesh
git merge --no-commit --no-ff main
```

5. Inspect the staged merge with `git status` and `git diff --cached`.
6. Either abort it or commit and push it.

### Practical result recorded

The test used the local line `Scenario 1: Dhruvesh WIP`.

- Before committing, the file appeared as `modified` and the line was only local working-tree data.
- After committing and pushing it to `origin/dhruvesh`, the line remained on the Dhruvesh branch.
- Local `main` was updated from GitHub.
- Switching back to `dhruvesh` showed the WIP line still present.
- `git merge --no-commit --no-ff main` automatically merged the incoming `main` additions and stopped before commit.
- `git diff --cached -- dhruvesh.txt` showed only the incoming `main` lines. The WIP did not appear in that diff because it was already a committed Dhruvesh change.
- The preview was accepted with a merge commit and pushed to `origin/dhruvesh`.

Conclusion: committed work on `dhruvesh` is not removed when updating and merging `main`. It is preserved or, if it conflicts with `main`, Git asks for a manual resolution.

## Scenario 2: work is pushed to `origin/dhruvesh` but has not reached `main`

### Situation

Dhruvesh has committed and pushed work:

```text
local dhruvesh == origin/dhruvesh
```

but that commit is not yet in remote `main`. Another developer's PR is merged into `main`.

### Result

Updating local `main` does not delete or move the `dhruvesh` branch. Switching back to `dhruvesh` returns to Dhruvesh's commits. A merge preview combines `main` with `dhruvesh`:

```text
origin/main        Other developer's reviewed changes
origin/dhruvesh    Dhruvesh's pushed, unmerged changes
       ↓
pending merge on local dhruvesh
```

The possible results are:

- Different files or lines: both changes remain.
- Same line: conflict; neither committed version is silently deleted.
- Preview rejected: `git merge --abort` removes only the pending merge state, not Dhruvesh's commits or the remote `dhruvesh` branch.

The practical Scenario 1 run also demonstrated this situation: the WIP commit was pushed to `origin/dhruvesh` before `main` was pulled and merged.

## Scenario 3: bad code was already merged into remote `main`

This scenario has not yet been practiced in this repository.

Once code is merged into shared `main`, do not erase history or force-push `main`. A correction requires a new commit and a PR.

For a normal bad commit:

```bash
git revert <bad-commit-sha>
git push origin dhruvesh
```

For a full PR merged through a merge commit:

```bash
git revert -m 1 <merge-commit-sha>
git push origin dhruvesh
```

Then create a new PR from `dhruvesh` to `main`. `git revert` preserves shared history while reversing the bad change.

## Team safety rules

1. `main` is the reviewed source of truth.
2. Keep the working tree clean before switching branches, pulling, merging, rebasing, or previewing a merge.
3. Commit important work to the permanent developer branch before integration. Push it when it needs remote protection.
4. Use `git pull --ff-only origin main` when synchronizing local `main`.
5. Use `--no-commit` merge previews to inspect incoming changes, then consciously abort or commit.
6. Never use `git reset --hard`, `git checkout -- <file>`, `git restore <file>`, merge strategy shortcuts, or plain force pushes as a way to dismiss legitimate conflicts.
7. If a particular file needs owner approval before entering `main`, configure GitHub `CODEOWNERS` and require code-owner approval in branch protection. This prevents an unapproved change from reaching `main`; it does not keep an old version after an approved new version is intentionally merged.