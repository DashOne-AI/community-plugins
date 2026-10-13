---
name: handoff
description: Generates (appends) a handoff entry to Handoff.md at the project root (goal, state, files touched, failed attempts and next steps) so work can resume in a fresh session after /clear. Manual invocation only. Works with requests in Spanish or English.
---

# Handoff: context transfer between sessions

## What it does

Writes a handoff entry at the end of `Handoff.md` in the **root of the current project**, with everything a new session (with no memory of this one) needs to resume the work. Afterward it tells the user to run `/clear` themselves. This skill CANNOT run `/clear` (it is a terminal command for the user, not a tool; do not attempt it through any channel).

## Procedure

### 1. Locate the file

- The file is always `Handoff.md` at the repo root (the git toplevel if the cwd is a repo, via `git rev-parse --show-toplevel`; otherwise the cwd).
- **Fixed name, no IDs or dates in the filename.** Each entry is identified in its internal header (step 3), not in the filename, so the new session always knows which file to open.

### 2. Gitignore (git repos only)

Check whether `Handoff.md` is already ignored (`git check-ignore Handoff.md`). If it is not, add a `Handoff.md` line to the root `.gitignore` (create it if it does not exist). This is internal working context and must never end up in a commit or a PR.

### 3. Append the entry

**Always append to the end of the file** (create it if it does not exist). Never overwrite or delete earlier entries. Parallel sessions of the same project share the same file, and the most recent entry is the last one.

Header of each entry:

```markdown
---

## Handoff - <YYYY-MM-DD HH:MM> · branch `<current branch>` · session <session id>
```

<!-- TRANSLATION NOTE: the original header used an em dash between "Handoff" and the date; replaced with a hyphen-minus to comply with the no-dash rule. -->

- Date/time: local, at the moment of writing (`date "+%Y-%m-%d %H:%M"`).
- Branch: `git branch --show-current` (omit the fragment if it is not a git repo).
- Session ID: the UUID of the current session if you know it (it appears in the session scratchpad path, e.g. `.../<uuid>/scratchpad`); if you do not have it, omit the fragment. Do not invent one.

### 4. Content: the 5 sections, in this order

```markdown
### 1) Goal
### 2) State at the end
### 3) Files and changes
### 4) Failed attempts
### 5) Next steps
```

**Time scope (important rule):**
- The **fine detail** (sections 3, 4 and part of 2) covers what happened **since the last `/compact`** of the conversation (or since the start if there was no compact).
- **Goal (1) and State (2)** are filled in using ALL the available context, including the compact summary. The goal was almost always established before the last compact, and it would be a mistake to leave it empty or invent it by limiting yourself to recent events.

Guide per section:

1. **Goal**: what is being attempted and why, in 1-3 sentences. The underlying "what for", not the latest micro-task.
2. **State at the end**: what is done and verified (with the command or evidence that proves it), what is half-finished, and any relevant external state (PRs opened or merged, deploys run or failed, pending secrets, running processes). Be honest: if something was assumed done without verification, say so.
3. **Files and changes**: list of paths touched (with the repo if more than one is involved) and one line per file saying what changed. Paths and description, NOT pasted diffs. Include commits and branches created, with their short hash.
4. **Failed attempts**: what was tried and did not work, and WHY (the diagnosis, not just "it failed"). This section exists so the new session does not repeat dead ends. It is the most valuable one; do not leave it thin. Also include discarded hypotheses with their evidence.
5. **Next steps**: an ordered, actionable list of what comes next, distinguishing what Claude can do alone from what is blocked waiting on the user (and exactly what is expected from them).

Write for a reader WITHOUT access to this conversation: no "the fix above", "as we saw", references to messages, or internal session codenames. Full paths, PR names with number and repo, exact commands.

### 5. Closing: tell the user

When finished writing, reply to the user with exactly this information (word it naturally, but keep it complete):

1. Confirmation that the entry is in `Handoff.md` (full path).
2. That they should now run `/clear` themselves.
3. That in the new session they should say: **"resume from Handoff.md"**. That is enough; the new session must read the entire file and treat the LAST entry as the current one (earlier ones are history, useful only as reference).

<!-- TRANSLATION NOTE: the original trigger phrase was in Spanish ("retomá desde Handoff.md"); translated to English. A Spanish-speaking user may use the Spanish equivalent ("retoma desde Handoff.md"), which should be treated the same way. -->

## When resuming (if this skill is invoked and a "resume from Handoff.md" request already exists)

This skill is for GENERATING the handoff. Resuming does not require the skill: read `Handoff.md`, take the last entry, verify against the real state of the repo (git status/log, PRs) that what it describes is still current (the repo may have moved between sessions), and continue from "Next steps".
