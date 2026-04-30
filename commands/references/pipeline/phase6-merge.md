### Merge worktree to master

Determine whether the work was done in a worktree by reading the `**Worktree:**` field from the impl plan, or by running `git worktree list` and checking if more than one entry exists.

**If in a worktree:**

Use AskUserQuestion to ask the user:

> "Pipeline complete -- all changes committed to the worktree branch. Ready to merge to master and delete the worktree? (yes/no)"

Set `MERGE_HAPPENED = false`.

- If **yes**:

  1. Use `WORKTREE_PATH` and `WORKTREE_BRANCH` captured in Phase 1.5. If these are unavailable, re-derive with:
     ```bash
     git worktree list
     ```

  1a. **Check for uncommitted main-repo changes before merging:**
     ```bash
     git -C "MASTER_REPO_PATH" status
     ```
     If any staged or unstaged changes exist in files that overlap with the worktree branch, **do not attempt the merge**. Instead, warn the user and offer two options:
     - **(a)** Commit the main-repo changes as a separate commit first, then rebase the worktree branch onto the new master, then proceed with the merge.
     - **(b)** Skip the automated merge and print the manual steps below.

     Do **not** attempt stash/pop automatically -- it creates conflicts that require human resolution.

  2. Rebase the worktree branch onto master (resolves any divergence before the merge):
     ```bash
     git -C "WORKTREE_PATH" rebase master
     ```
     If the rebase has conflicts: describe every conflicting file to the user and stop. Tell them to resolve the conflicts inside the worktree and then run the following manually when ready:
     ```
     ExitWorktree action="keep"   (if the pipeline session called EnterWorktree)
     git -C "MASTER_REPO_PATH" merge --ff-only WORKTREE_BRANCH
     git worktree remove "WORKTREE_PATH"
     git -C "MASTER_REPO_PATH" branch -d WORKTREE_BRANCH
     git -C "MASTER_REPO_PATH" worktree prune
     ```

  2a. Clear the worktree session state so the harness no longer tracks this worktree:
      Call `ExitWorktree action="keep"`.
      This returns the session CWD to the main repo. The worktree directory
      still exists on disk (cleaned up in step 4), but the harness no longer
      considers itself "inside" it. Without this step, the harness session
      state becomes stale after step 4 deletes the directory, causing
      "path does not exist" errors in the next conversation.

  3. Merge into master (fast-forward only -- guarantees linear history):
     ```bash
     git -C "MASTER_REPO_PATH" merge --ff-only WORKTREE_BRANCH
     ```
     If this fails (non-fast-forward): tell the user the rebase did not produce a clean fast-forward and they must resolve it manually.

  4. Clean up the worktree:
     ```bash
     git worktree remove "WORKTREE_PATH"
     git -C "MASTER_REPO_PATH" branch -d WORKTREE_BRANCH
     git -C "MASTER_REPO_PATH" worktree prune
     ```

  5. Set `MERGE_HAPPENED = true`. Report: "Merged to master and worktree deleted. Run `git push` when ready to push to remote."

- If **no**:
  Set `MERGE_HAPPENED = false`.
  Report the manual steps so the user can do it later:
  ```
  git -C "WORKTREE_PATH" rebase master
  ExitWorktree action="keep"   (if the pipeline session called EnterWorktree)
  git -C "MASTER_REPO_PATH" merge --ff-only WORKTREE_BRANCH
  git worktree remove "WORKTREE_PATH"
  git -C "MASTER_REPO_PATH" branch -d WORKTREE_BRANCH
  git -C "MASTER_REPO_PATH" worktree prune
  ```

**If not in a worktree:**

Check for uncommitted changes with `git status`. If any exist, ask the user if they want to commit them. If everything is already committed, skip silently.

Set `MERGE_HAPPENED = true`.

### Update patch notes

**Only run if `MERGE_HAPPENED = true`** (skip if the user declined the merge).

Check whether `package.json` exists at the project root. If it does, invoke the patch-notes skill:

```
Use the Skill tool with skill: "patch-notes"
```

This records every new commit since the last patch notes entry, generates AI-written summaries, bumps the version in `package.json`, and commits `PATCHNOTES.md` + `package.json` automatically. No user input required.

If `package.json` does not exist, skip this step.
