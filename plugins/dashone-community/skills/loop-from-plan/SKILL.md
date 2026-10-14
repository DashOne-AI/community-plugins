---
name: loop-from-plan
description: Takes an approved plan and launches its autonomous execution in one move. It builds the tailored execution-authorization prompt (via loop-instructions-builder's analysis), ensures every phase/slice has a matching GitHub issue (cross-repo issues with cross-reference when a slice touches more than one repo), and immediately starts /loop with that prompt, using the plan's acceptance criteria as the loop's termination condition. Works with requests in Spanish or English. Use when the user wants to go from "plan approved" straight to "running in a loop" without manually copy-pasting the generated prompt into /loop, or without manually creating tracking issues first. Trigger on requests like "ejecuta este plan en loop", "arranca el loop con este plan", "corre el plan de forma autónoma hasta que se cumplan los ACs", "/loop-from-plan", or the English equivalents. Takes an optional interval for /loop (e.g. "10m"); omitted means self-paced.
---

# Loop From Plan

## What this is

`loop-instructions-builder` produces an execution-authorization prompt and stops, and the user then pastes it into `/loop` by hand. This skill removes that manual hop: same analysis, same prompt quality, but the output is **fed directly into `/loop`** and execution starts immediately. It also guarantees the plan is tracked as GitHub issues before any code gets touched (one issue per phase/slice, cross-repo issues cross-referenced when a slice spans more than one repo), and wires per-iteration issue verification into the loop prompt itself.

Pipeline: find plan → analyze blockers → ensure one GitHub issue per phase/slice (cross-repo if a slice touches more than one repo) → build prompt → adapt for loop execution (embeds issue-verification instructions) → invoke `/loop`.

## Step 1: Load the builder and run its analysis

Invoke the `loop-instructions-builder` skill (Skill tool) and follow its Steps 1 to 3 **fully and freshly**:

1. Find the real plan (active plan-mode file, pasted plan, or ask; a one-liner is not enough).
2. Check every step against Claude Code's hard limits (hard-blocked / needs-specific-ask / regular). Do not skip this or do it from memory. The entire value is re-deriving what's blocked for *this* plan.
3. Write the authorization prompt with the builder's fixed shape (authorized list, hard-blocked carve-out if any, sequencing constraints if real, stop-and-flag block, testing-rigor instruction, anti-blanket clause, closing summary request).

Do NOT execute the builder's Step 4 (deliver as copy-paste block). This skill replaces delivery with execution.

## Step 2: Ensure one GitHub issue per plan phase/slice

Each phase or slice in the plan is a unit of work with its own issue. This is what makes progress visible per-slice (to the user, to a GitHub Project board, to a future session) instead of one opaque "implement the plan" blob. Do this once, right after Step 1's blocker analysis, before building the loop prompt.

1. **Confirm `gh` is usable.** Run `gh auth status`. If it fails, tell the user `gh` isn't authenticated and stop before creating anything or launching the loop. Do not silently skip issue creation and proceed as if this step succeeded.

2. **Identify the main repo.** The repo of the plan's primary working directory: `gh repo view --json nameWithOwner -q .nameWithOwner`. If that directory isn't a git repo yet, stop and tell the user (issues need a repo to attach to), and offer to `git init` per the standing guidance on that, but don't do it unilaterally.

3. **Break the plan into phases/slices.** Use the plan's own phase/step structure; plan-mode plans are typically already sliced into steps or milestones. If the source is a flat task description instead, derive natural slices from it at the granularity a human would PR separately. Each slice = exactly one issue in the main repo, at minimum.

4. **Detect cross-repo slices.** For each slice, check whether its steps touch paths outside the main repo: other project directories among this session's additional working directories, other repo names/paths the plan names explicitly, or submodules of the main repo (`git submodule status`) whose path the slice's steps touch. A slice confined to the main repo needs one issue; a slice touching N repos needs one issue per repo it touches.

5. **Search before creating, never duplicate.** For each (slice, repo) pair, search first: `gh issue list --repo <owner>/<repo> --state all --search "<slice title keywords>"`. If an existing issue already represents this slice, reuse it (record its number) instead of creating a new one.

6. **Create missing issues, cross-referenced when multi-repo:**
   - Create the main-repo issue first: `gh issue create --repo <owner>/<repo> --title "[Phase N] <slice name>" --body "<slice description + this slice's own acceptance criteria, not the whole plan's>"`.
   - For each secondary repo the slice touches: `gh issue create --repo <owner>/<repo2> --title "[Phase N] <slice name>" --body "<same, plus a line: 'Part of: <owner>/<repo>#<main-issue-number>'>"`.
   - Then comment back on the main issue listing every secondary issue just created: `gh issue comment <main-issue-number> --repo <owner>/<repo> --body "Sub-issues: <owner>/<repo2>#<N2>, ..."`.
   - Creating the main issue first, then referencing its number from the secondary issues, then commenting back, is what makes GitHub's native cross-linking work in both directions without having to guess a number before it exists.

7. **Record the mapping.** Build a slice → issue(s) table (`repo#number`, one row per slice, listing every repo it touches). Step 3 embeds this table directly in the loop prompt so every iteration can check against it.

Do not move anything to "In Progress" yet in this step. That happens when work actually starts on a slice, which is the loop's job per-iteration (Step 3 below), per the standing "mark In Progress when starting" rule, not this one-time setup step.

## Step 3: Adapt the prompt for /loop

The builder's prompt is written as a one-shot authorization. A /loop prompt re-fires every iteration, so add these adaptations on top of the builder output:

1. **Acceptance criteria as termination condition.** Extract the plan's acceptance criteria (or its "what done looks like" section; if the plan has neither, derive observable ACs from the plan's steps and list them explicitly). Append a block:

   ```
   ### Acceptance criteria (loop termination condition)
   - [AC 1]
   - [AC 2]
   ...
   The loop ends ONLY when: (a) every AC is verified with real evidence (not
   assumed), or (b) everything remaining is blocked waiting on the user.
   In either case: stop the loop (ScheduleWakeup with stop: true) and deliver
   the closing summary.
   ```

2. **Iteration semantics.** State that each iteration must: re-read current state (do not trust memory of a prior iteration, since the repo, deploys, and DB may have moved), pick up the next unfinished/unblocked item, work it to a verified state, and update progress. Blocked items go in the BLOCKED block and stay visible every iteration until resolved; unblocked work continues in parallel.

3. **No-progress guard.** If two consecutive iterations make zero progress on the same item for the same reason, stop iterating on it, mark it BLOCKED with the honest diagnosis, and move on, or stop the loop entirely if nothing else remains.

4. **Issue-per-phase verification.** Embed Step 2's slice → issue mapping directly, and require every iteration to check against it before touching code:

   ```
   ### Issues per phase (verify every iteration, before touching code)
   - Phase 1, <title>: <owner/repo>#<N>
   - Phase 2, <title>: <owner/repo>#<N> (cross-repo: <owner/repo2>#<M>)
   ...

   Before touching code in any iteration: confirm the phase you are about to work
   on has an issue in the list above. If the phase has no issue yet (it came up
   after this prompt was built, or the plan's scope grew), create it before
   continuing, with the same rule used when the plan was built: 1 issue in the
   main repo, +1 issue per additional repo the phase touches, cross-referenced
   with each other. When you start working on a phase, move it to "In Progress"
   (global Project status rule, if the issue is linked to one). When you complete
   and verify a phase, close its issue(s) with a short comment containing the
   verification evidence.
   ```

5. **Agent per step.** The main session orchestrates; each phase/slice is delegated by name to a subagent (Agent tool, `subagent_type`). Assign the agent when building the Step 2 mapping and embed it in the prompt:

   Use the agents your setup defines. If it defines none, use `general-purpose` for every step and treat the two tiers below as one.

   - **Default tier** (a fast, cheaper agent): the what and the where are already decided; edit, write tests, run the gate. This is the default.
   - **Escalation tier** (a stronger agent): the step is open-ended or hard (large migration, multi-file refactor, diagnosis with several hypotheses), or a default-tier agent already failed the gate twice on that step.
   - No delegation (the main session does it): design decisions, merges, anything that requires the user's judgment.

   ```
   ### Agent per phase (delegate by name; escalate on failure)
   - Phase 1, <title>: <default agent>
   - Phase 2, <title>: <escalation agent> (<reason>)
   ...

   Delegate each phase to the indicated agent. If a default-tier agent reports 2
   consecutive gate failures on the same phase, do not retry with another
   default-tier agent: reassign that phase to the escalation agent, passing it what
   was tried and why it failed. If the escalation agent also fails, mark the phase
   BLOCKED with the diagnosis.
   ```

   If your setup has a hook or rule that warns after repeated gate failures, route the escalation to the escalation agent. A slash command the user has to type, such as a manual model switch, cannot run inside an autonomous loop.

Keep everything else from the builder's prompt verbatim (carve-outs, sequencing, anti-blanket clause). Language: the three template blocks above are written in English. If the plan or conversation is in another language, translate the blocks into that language before embedding them in the prompt.


## Step 4: Launch /loop

Invoke the `loop` skill (Skill tool) with args = `[interval] <adapted prompt>`:

- If the user passed an interval to this skill (e.g. `/loop-from-plan 10m`), prepend it.
- Otherwise omit the interval. Self-paced dynamic mode is the default and correct for plan execution (pace matches the work, not a timer).

Before launching, show the user, **outside** any code block, a summary covering: what's authorized, what's carved out as theirs (if anything), the issues created or reused per phase (`repo#number` list, flagging any cross-repo pairs), and the ACs that will stop the loop. Then launch immediately; do not ask for confirmation (the user invoked this skill precisely to skip that hop). Exception: if Step 1's analysis found a hard-blocked step that the ENTIRE plan depends on, or Step 2 could not get a usable main repo (`gh` unauthenticated, target not a git repo), don't start the loop. Surface the blocker first and stop.

## Failure modes to avoid

- Reusing a prior run's authorization wording instead of re-analyzing this plan.
- Padding a blockers section when the plan has zero hard-blocked steps. The builder's rule applies: say "full autonomy" and move on.
- Launching a loop whose prompt has no termination condition. ACs are mandatory in the adapted prompt; a loop that can't tell when it's done never stops.
- Delivering the prompt as text and stopping. That's `loop-instructions-builder`'s job, not this skill's. This skill ends with /loop running (or a surfaced total blocker).
- Creating a duplicate issue for a slice that already has one. Always search (`gh issue list --search`) before creating.
- Creating a secondary-repo issue without cross-referencing it back to the main-repo issue (or vice versa). An orphaned issue that doesn't say which slice or plan it belongs to defeats the point of tracking.
- Treating issue creation as optional or "nice to have if there's time". Every slice needs its issue(s) before the loop launches, not retrofitted after the fact once code is already moving.
- Launching the loop without embedding the slice → issue mapping in its prompt. Without it, no iteration has anything concrete to verify against.
