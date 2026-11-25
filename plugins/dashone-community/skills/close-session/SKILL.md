---
name: close-session
description: Close out a work session safely. Finds what the session left open (uncommitted code, open PRs, stale branches, open GitHub and Linear issues, unindexed artifacts), commits and pushes code, merges green PRs, closes issues and finished epics, deletes merged and session-owned branches, registers new artifacts in the Project Index, reports, and archives the session. Use when the user says "close the session", "wrap up", "clean up", "cierra la sesión", "cerrar sesión", "ya terminé" or "limpia todo", or at the end of a work session.
argument-hint: "[--dry-run]"
---

# Close Session

Closes a session in one pass. The goal is that the user comes back to nothing half-open: no orphan branch, no unpushed code, no PR waiting, no issue that should be closed, no artifact missing from the index.

## Usage

```
/dashone-community:close-session
/dashone-community:close-session --dry-run
```

`--dry-run` runs steps 1 and 2 only, prints the plan of actions, and changes nothing. Recommend it the first time the skill is used in a new repo.

## Golden rule: the ledger

Only things this session created or touched are changed automatically. Build the **ledger** first from the conversation and the tool history:

- Branches this session created or checked out and pushed.
- PRs this session opened or subscribed to.
- GitHub issues this session created or referenced with `Closes` or `Refs`.
- Linear issues this session created, commented on or moved.
- Artifacts and docs this session published.
- Every repo this session worked in: the working directory and any repo attached to the session.

Anything not in the ledger is listed to the user and left alone unless they say otherwise. When the ledger and reality disagree, trust reality (git, GitHub, Linear) and say so.

## Safety rails (always)

- No force push, no history rewrite, no `git reset --hard`, no `git clean`.
- Never delete the default branch, `staging`, or the branch that is currently checked out (switch to the default branch first).
- `git branch -d` by default. Use `-D` only for a ledger branch whose PR is merged (squash) or closed and whose tip equals the PR's final head SHA (step 7), and say so in the report.
- Never commit `.env`, credentials, tokens, keys, local settings files or large binaries. If a changed file looks like a secret, stop and ask. Never print secret values.
- Repo rules win: read the repo's contributor instructions (for example `CLAUDE.md`) before committing or merging: branch naming, commit message format, merge method, deploy behavior.
- Unexpected state (detached HEAD, merge in progress, a branch someone else pushed to, a PR from another author): stop, list it, ask.
- Never mark anything done that was not verified. Report what could not be verified.

## Steps

Run in this order. Order matters: PRs before branches, report before archive.

### 1. Build the ledger
As described above. Keep it short: one line per item with its link.

### 2. Inventory (read only)
For each repo in the ledger:

- `git status -sb`, `git branch -vv`, `git log @{u}..HEAD` for unpushed commits. In a shallow clone the remote-tracking ref may be missing: `git ls-remote --heads origin <branch>` is the truth, and `git fetch origin +refs/heads/<b>:refs/remotes/origin/<b>` creates the ref.
- Open PRs (GitHub tools) and, for ledger PRs, CI state and mergeability.
- Open ledger issues on GitHub and in Linear.

Print one table: item, state, planned action. With `--dry-run`, stop here and finish with the report in step 9 (marked as dry run). Do not archive.

### 3. Uncommitted code
For each repo with changes in the ledger scope:

1. Run the repo's fast checks on what changed (tests, typecheck). Failing checks: stop and ask, do not commit broken code silently.
2. Commit on the session branch (create one if on the default branch, following the repo's naming rule). The message follows the repo convention and references the issue (`Closes #N` or `Refs #N`). Add any commit attribution the session or repo requires.
3. Push with `git push -u origin <branch>`, retrying network errors up to 4 times with backoff. If the push is rejected because the remote has commits this session lacks, fetch and merge, do not force.
4. Open a PR if none exists for the branch (mirror the repo's PR template headings). It then enters step 4.

Decide whether anything still needs pushing: a branch with commits ahead of its remote always does.

### 4. Pull requests
For every open PR in the ledger:

- CI green, no conflicts, no unresolved review threads: **merge** with the repo's method (read its contributor instructions; for example squash, or a merge commit for a staging to main promotion).
- Anything else (red CI, conflict, open threads, approval needed): do not merge. List it with the reason and ask.
- Merging may deploy to production (for example repos whose CI deploys on every push to `main`). Always state in the report which merges triggered a deploy and where to watch it.
- After a merge, stop watching that PR (unsubscribe from its activity if the session subscribed).

PRs open in the repo that are not in the ledger are listed, never touched. In a repo with many old PRs this list is long: show the count and the three most recent, not all of them.

### 5. GitHub issues
For each ledger issue: confirm it is closed. `Closes #N` auto-closes only when it is at the top level of the PR body, not inside a code block. If it is still open and its PR merged, close it with a final comment that names what shipped and links the merged PR. Use `state_reason: completed`. End every GitHub comment with any footer the session instructions require.

### 6. Linear
Use the Linear tools (`get_issue`, `save_issue`, `save_comment`).

- Ledger issue whose PR merged and acceptance is met: set state Done and add a comment with the merged PR link.
- Ledger issue whose PR is open or whose work is partial: leave open, add a comment with the current state.
- **Epics** (a parent issue with sub-issues): close only when every child is Done or canceled. Re-read the children after closing the ones from this session. If one child is still open, leave the epic open and name that child.
- Linear **projects** are reported, never closed by this skill.
- Write comments in the language the project already uses (check its labels and existing issues).
- No match after one miss: try the alias, the client name, then free-text search before saying it does not exist.

### 7. Branches
Only after step 4, so merged PRs are accounted for.

1. `git fetch --prune origin`.
2. For each repo, candidates are: branches whose PR is **merged or closed** (ask GitHub by head branch), and branches that are plain ancestors of the default branch. Do not rely on ancestry alone: squash merges leave the branch tip outside the default branch history, so `git branch --merged` and `git merge-base --is-ancestor` say "not merged" for work that shipped.
3. Before deleting a squash-merged or closed branch, check that its tip equals the PR's final head SHA (nothing was pushed after the merge). If the tip is ahead of the PR head, there is unshipped work: do not delete, list it.
4. Delete the local branch and the remote branch (`git push origin --delete <branch>`; skip if GitHub already auto-deleted it). Use `git branch -d` for true ancestors. For a squash-merged or closed ledger branch whose tip matches the PR head, `git branch -D` is required (git refuses `-d`); that is the one allowed use of `-D`, and the report says so.
5. Never delete a branch with unmerged commits and an open PR, or anything outside the ledger that is unmerged. List those instead.
6. `git remote prune origin` at the end.

### 8. Project Index
If the ledger has artifacts or docs created in this session and the `project-management` plugin is installed, update the project's index: invoke its `project-index-manager` skill, section "Register what is new", for each one (find the index first; ask the user for the link only if it cannot be found). Skip silently when the ledger has no artifacts or the plugin is not installed. Never invent metadata: unknown fields stay empty.

### 9. Report
Visible text, in the user's language, terse, with links. One short block per area, only areas with something in them:

- Merged (PR links) and which of those triggered a deploy.
- Closed (GitHub and Linear issues, epics).
- Branches deleted (names).
- Committed and pushed (commit and PR).
- Indexed (artifact titles).
- **Left open and why** (needs the user): each item with the exact next step.
- **Not verified**: anything that could not be checked.

### 10. Archive the session
Last action of the skill.

- Archive only when nothing is blocked and nothing waits on an answer from the user. If anything is left open, skip the archive, say so in one line, and ask whether to archive anyway.
- `--dry-run`: never archive.
- In a Claude Code cloud session, get the session id (the session lookup tool with no `session_id`), then archive it with the session archive tool. The session becomes read-only and its container is released, so write the report (step 9) first and do nothing after the call. Unarchiving reverses it.
- If the archive tools are not available, say the session was not archived and that the user can archive it from the app.

## Notes

- Do not open new work. This skill closes loops, it does not start any.
- Do not narrate each tool call. Report results.
- When asked "what is left?" mid-session, run steps 1 and 2 only.
