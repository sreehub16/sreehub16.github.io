# GEI Migration Failure — 2GB Pack Size Limit: Remediation Runbook

## Overview
This runbook helps identify and resolve migration failures caused by GitHub's 2GB pack-size limit during ADO → GitHub Enterprise Cloud migrations using GEI (`gh ado2gh migrate-repo`).

**Error signature to watch for:**
```
[ERROR] Git source migration failed. Error message: Push error: Errors reported by remote:
fatal: pack exceeds maximum allowed size (2.00 GiB)
error: remote unpack failed: index-pack failed
References rejected by remote:
refs/github-services/chunked-upload/<COMMIT_SHA> => failed
```

---

## Step 1: Create a Backup Copy of the Repository

Before touching the production repo, create a full working copy in ADO with all branches and commit history intact.

### Try the CLI mirror clone/push first
```powershell
git clone --mirror https://dev.azure.com/<org>/<project>/_git/<original-repo>
cd <original-repo>.git
git push --mirror https://dev.azure.com/<org>/<project>/_git/<backup-repo-name>
```
> The target repo (`<backup-repo-name>`) must already exist (empty) in ADO before pushing.

### Option: CLI mirror with only `main` (skip other branches/tags)
If you only need `main` (not the full set of branches/tags), this keeps the push smaller and may avoid the 5GB limit entirely:
```powershell
git clone --single-branch --branch main https://dev.azure.com/<org>/<project>/_git/<original-repo>
cd <original-repo>
git remote set-url origin https://dev.azure.com/<org>/<project>/_git/<backup-repo-name>
git push origin main
```
This clones `main` with its **full commit history** (not shallow) and pushes just that branch — no tags or other branches come across.

### If either option above still fails with a 5GB push-size error
```
TF402462: This push was rejected because its size is greater than the 5120 MB limit for pushes in this repository.
```
This is a **hard, non-configurable ADO limit** — there is no admin setting to raise it. Use ADO's web-based import instead:

1. Create the empty target repo (`<backup-repo-name>`) in ADO.
2. Go to the new repo → **Files** → **Import repository**.
3. Set the source URL to the original repo:
   `https://<org>.visualstudio.com/<project>/_git/<original-repo>`
4. This replicates the full mirror (all branches, tags, history) server-side, bypassing the client push-size ceiling.

> Note: ADO's import feature always imports the entire repo (all branches/tags) — there's no option to import only `main`. If you only need `main`, delete unwanted branches afterward:
> ```powershell
> git push origin --delete <branch-name>
> ```

---

## Step 2: Identify the Offending Commit

Run the GEI migration against the **backup repo** first. When it fails with the pack-size error, the failing commit's SHA is embedded directly in the log:

```
refs/github-services/chunked-upload/<COMMIT_SHA> => failed
```

This SHA has been confirmed (via testing) to point to the actual commit responsible for the oversized push — not just the branch tip.

---

## Step 3: Verify the Commit's Size

### List files changed in that commit
```powershell
git diff-tree --no-commit-id -r <COMMIT_SHA>
```
This shows each file's blob SHA and path.

### Get exact byte sizes per file
```powershell
git diff-tree --no-commit-id -r <COMMIT_SHA> | ForEach-Object {
    $parts = $_ -split '\s+'
    $blob = $parts[3]
    $path = $parts[5]
    $size = git cat-file -s $blob
    "$path : $size bytes ($blob)"
}
```

### Confirm total pack size
```powershell
git gc
git count-objects -vH
```
Check the `size-pack` value — if it's at or above ~2GB, this confirms the commit needs to be split.

> Tip: watch for duplicate blob SHAs across files (identical content stored once by git) — these don't add extra pack size even though they appear as separate files.

---

## Step 4: Split the Oversized Commit

Group the files from Step 3 into batches that stay safely under the 2GB ceiling — **target ~1.5GB per commit** to leave headroom for git/pack overhead.

### Start the rebase
```powershell
git rebase -i <COMMIT_SHA>~1
```
In the editor, change `pick` to `edit` on the line for `<COMMIT_SHA>`, then save and close.

### Undo the commit, keep files staged
```powershell
git reset HEAD~1
```

### Re-commit in smaller batches
```powershell
git add <file-group-1>
git commit -m "part 1 of large files"

git add <file-group-2>
git commit -m "part 2 of large files"

git add <file-group-3>
git commit -m "part 3 of large files"
```
(Add as many grouped commits as needed to keep each batch under ~1.5GB.)

### Continue the rebase
```powershell
git rebase --continue
```

> **If rebase reports a conflict with untracked files:** this usually means the local branch wasn't fully up to date before starting. Sync fully first (`git rebase --abort` to reset, then re-sync, then restart from the top of Step 4) rather than deleting the conflicting files — they may contain content you need.

### Verify the split
```powershell
git log --oneline
git gc
git count-objects -vH
```

---

## Step 5: Push the Rewritten History

Since this rewrites commit SHAs from the split point forward, a force push is required:

```powershell
git push --force-with-lease origin main
```

> **Avoid using your IDE's "sync" feature after a rebase.** If the force push fails (e.g., due to a branch policy) and you let a sync/pull run instead, it can silently merge the old and new histories back together, reintroducing the oversized commit. If this happens, reset to your last known-good rebased commit and force-push again:
> ```powershell
> git reset --hard <last-good-commit-sha>
> git push --force-with-lease origin main
> ```

If the force push is blocked entirely, check ADO **Branch Policies** for the target branch — force pushes to protected branches may need to be temporarily allowed by an admin.

---

## Step 6: Re-test the Migration

Run the GEI migration again against the **backup repo**:

```powershell
gh ado2gh migrate-repo --ado-org <org> --ado-team-project <project> --ado-repo <backup-repo-name> --github-org <github-org> --github-repo <target-repo-name>
```

- **If it succeeds:** the fix is validated. Proceed to Step 7.
- **If it fails again** with the same pack-size error: repeat Steps 2–5, since there may be more than one oversized commit in the history.

---

## Step 7: Apply the Fix to the Production Repo

Once validated on the backup copy:

1. Repeat Steps 2–5 on the actual production repo.
2. Keep the original ADO production repo untouched/available as a rollback point until the GitHub migration is confirmed successful.
3. Run the GEI migration against the production repo.

---

## Quick Reference Summary

| Step | Action | Key Command |
|---|---|---|
| 1 | Backup repo | `git clone --mirror` / ADO web import |
| 2 | Identify failing commit | Read SHA from `chunked-upload/<sha>` in migration log |
| 3 | Verify commit size | `git diff-tree` + `git cat-file -s` |
| 4 | Split commit | `git rebase -i <sha>~1` → `edit` → `reset HEAD~1` → re-commit in batches |
| 5 | Push rewritten history | `git push --force-with-lease` |
| 6 | Re-test on backup | Re-run GEI migration |
| 7 | Apply to production | Repeat on real repo once validated |
