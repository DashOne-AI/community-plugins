---
name: plan-multi-agent
description: Plan any piece of work end to end and leave it ready to execute, by one agent or by a supervisor with several agents in parallel. On Opus 5.5 or Fable only, in plan mode. It runs `diagnose` first when the request is a problem, asks 3 to 10 gap-closing questions, and reads every issue in full when the input is a backlog, checking each against the code so nothing already done gets planned. It splits the work into units with dependencies, owned files, acceptance criteria, a production-write register and a merge gate, and minimizes what the human has to do. It decides whether execution is normal (sequential) or multi-agent (parallel) and checks the plan against a readiness checklist. After approval it creates the Linear issue, sub-issues and plan document, plus the GitHub ledger, and hands off to loop-from-plan, which reads the execution mode from the plan. Works in Spanish or English. Use when the user says "/plan-multi-agent", "/linear-plan", "planea esto", "arma el plan", "haz un plan para…", "planéalo en Linear", "planea todos los issues abiertos", "arma un plan para varios agentes en paralelo", "plan this", "make a plan for…", "plan this in Linear", "plan this backlog for parallel agents", or asks to turn an idea, feature, bug, incident or backlog into an executable plan with tickets. Do NOT use to implement the plan; hand that to loop-from-plan once the user approves.
argument-hint: "[idea, problem, or scope: a Linear project, a GitHub label, a list of issues]"
---

# plan-multi-agent

Idea, problem or backlog → closed gaps → units with acceptance criteria, dependencies and owned
files → the execution mode decided (normal or multi-agent) → Linear issue, sub-issues and plan
document, plus the GitHub ledger → handed to `loop-from-plan` with as little human work as possible.

A plan one person reads is not the same as a plan a supervisor and four agents execute without
asking questions. The failure modes are known:
- two agents edit the same file;
- a unit counts as "done" because the tests passed and nobody walked the journey;
- an agent applies a migration it should have asked about;
- the plan lives in a session that is gone by tomorrow.

This skill avoids them and does not call a plan ready until the readiness checklist passes.

All output goes in the language the user used. If that is Spanish, use neutral Latin American
Spanish. Follow the repo's own conventions (`CLAUDE.md`, `AGENTS.md`, contributing guide)
wherever they exist: branch naming, issue rules, how "done" is proven, what deploys on merge.

## Step 0: Opus 5.5 or Fable, and plan mode (before anything else)

A skill's frontmatter cannot change the model or the permission mode; do it with tools.

1. **Model gate.** Allowed: Opus 5.5 (`claude-opus-5-5`) or Fable (`claude-fable-5-*`). Check
   which model you are running as (your system prompt, or `get_session` if the session tools exist).
   - Allowed model: say nothing and continue.
   - Otherwise, if `mcp__ccd_session_mgmt__set_session_model` exists (load it with ToolSearch),
     set `claude-opus-5-5` for this session and, if `set_session_effort` exists, effort `high`
     or above. Continue only once the switch took effect.
   - If no tool can switch it, tell the user in one line: "Switch to Opus 5.5 (`/model opus`) or
     Fable and run `/plan-multi-agent` again", and stop. Do not plan with another model.
2. **Plan mode.** Call `EnterPlanMode` (load it with ToolSearch if it is deferred). If the user
   rejects it, continue in strict read-only mode until the plan is approved: no Edit/Write
   outside the plan file, no commands that write, deploy, migrate or spend, no Linear or GitHub
   writes.

## Step 1: Problem, build or backlog?

Classify the request:

- **Problem**: something is broken, wrong, slow, failing or "doesn't work" (a bug, an alert, an
  incident, a regression, data that looks off).
- **Build or change**: a feature, an improvement, a migration, a refactor or a new project.
- **Backlog**: a set of existing issues to clear (a Linear project, a label, a list of issues).

If it is a **problem**, run the `diagnose` skill first (`diagnose`, or
`dashone-community:diagnose` when installed as a plugin) through its Steps 1 to 3: context,
read-only investigation with evidence, standard analysis. Do **not** stop at its Step 4
`ExitPlanMode`. Fold its report into this plan as the **Diagnosis** section and carry its
priority, effort and fix plan forward into Step 5. Do not re-ask questions `diagnose` already asked.
If `diagnose` is not available, apply its rules inline: identify the real process, build and
config that produced the symptom with real observability, and mark every hypothesis as
confirmed, ruled out or open. Never plan a fix on an unconfirmed cause.

If it is a **backlog**, run the inventory in Step 2b before asking anything.

## Step 2a: Find the Linear destination

Infer it; do not ask first:

1. The repo: mentions of a Linear team or project in `CLAUDE.md`, `AGENTS.md` or `README.md`, the
   git remote, and existing Linear issues that reference the repo (`list_issues` with a keyword
   query).
2. The client or project context of the session: a client-context skill if one is available,
   the working directory, the client named in the request.
3. Existing Linear projects that match (`list_projects`), so the work lands in one that already
   exists instead of a new one.

If the team or project is still ambiguous, it becomes one of the Step 3 questions. Statuses and
labels are per team: read them with `list_issue_statuses` / `list_issue_labels` before using
them.

## Step 2b: Backlog inventory (only for a backlog)

1. **Read every issue in full** (description, relations, comments), not the titles. For Linear
   use `list_issues` then `get_issue`; for GitHub use the issue tools or `gh`. Record its status,
   priority, parent, duplicates and linked pull requests.
2. **Verify each issue against the code, today.** Decide exactly one of, with evidence (file and
   line, or a PR number): `DONE`, `PARTIAL`, `NOT STARTED`, `DUPLICATE of X`, `DEPENDS ON X`,
   `OTHER REPO`, `NOT CODE`.
   - Check `git log` and the pull requests merged since the issue was written.
   - When a flag or a migration is involved, check production with a read-only query, never a write.
   - The audits can run as parallel read-only agents, one per cluster of issues.
   - Date the audit, because each unit re-checks its own premise when it starts.
   - When an issue's description contradicts the code, the code wins; say so in the plan.
3. **Plan the closure of what is already done.** List the issues to move to Done, Duplicate or
   out of the project, each with its evidence. Plan no work for them. Apply those changes in
   Step 8, after approval, not while planning.

## Step 3: Close gaps, at least 3 questions and at most 10

Always ask at least 3 questions, and stop at 10. Use `AskUserQuestion` (up to 4 per call, so up
to 3 rounds), or a short text list when the answers are free-form. Each question must stand alone
and name what it is about.

- Never ask what you can verify yourself (code, logs, git, Linear, docs, database reads). Verify
  it, then ask about what is left.
- Prioritize the gaps that change the plan the most:
  1. **Outcome and done**: what the user observes when it is finished.
  2. **Scope**: what is explicitly out of scope.
  3. **Constraints**: deadline, budget, stack, environments, things that must not change.
  4. **Human touchpoints**: which accesses, accounts, approvals and secrets exist, and who holds
     them (this feeds Step 5).
  5. **Linear destination**, if Step 2a did not resolve it.
  6. **Rollout**: staging first, feature flag, who must be told.
  7. **Execution** (when the work could run in parallel): whether to run it with several agents,
     the concurrency cap (default 4), branches per unit, whether the supervisor may merge on its
     own when the merge gate passes, and which production writes always need the user.
- For a backlog, the same rounds can carry the triage of each unit: yes, adjust, no or later,
  plus its priority.
- Stop asking once the answers stop changing the plan, but never before 3.
- When the user answers "I don't know", continue and record it as an assumption in the plan.

## Step 4: Size, units and execution mode

**Size.** Estimate the effort with the `diagnose` scale: **XS** < 1 h · **S** 1 to 4 h · **M** ½
to 2 days · **L** 2 to 5 days · **XL** > 1 week. The work is **epic size** when it is L or XL,
spans more than one repo or service, or needs more than one independently shippable change.

**Units.** Split epic-size work, and every backlog, into **units**. A unit is a phase in plain
plans:
- it ships and is verified on its own, with its own acceptance criteria;
- it is one pull request that one agent can finish alone;
- it is no larger than about a day of agent work, with at most one migration.

Group a backlog by **root cause and file ownership**, not by issue title, and name the duplicates
each unit absorbs. Order the units so the one that removes the most risk comes first. When later
units need a test harness or fixtures as their gate, that unit goes first. Non-epic work is a
single unit.

**Dependency graph.** Give each unit `depends_on`, with no cycles. When two units touch the same
file, either order them, or restrict the shared file to a single dispatch line or schema entry and
put the logic in a new module. Group the units into waves for the concurrency cap.

**Execution mode.** Choose one and write it in the plan marker:
- **multi-agent**: at least two units can run at the same time (no dependency between them, no
  shared owned file) and the user wants autonomous execution;
- **normal**: anything else. That includes a single unit, a chain where every unit depends on the
  previous one, and a run the user wants done by one agent.

Never claim parallelism that the dependencies and owned files have not proven.

## Step 5: Execution strategy, with as little human as possible

Default to the agent doing the work. Classify every step:

- **Agent**: code, tests, reads, PRs, Linear and GitHub updates, and verification with the tools
  it has. Most steps.
- **Human, hard-blocked**: things the agent cannot do whatever it is authorized to do. Entering
  passwords or secrets, OAuth or consent screens, 2FA or OTP codes, pairing a device (a QR scan),
  payments, account-owner approvals, security or system settings.
- **Human, by policy**: things the repo rules reserve for a person, such as a merge that needs a
  human review or a message to a client.
- **Production write**: a migration, a backfill, a flag, a secret, a deploy, or a message sent on
  the user's behalf. It goes in one **production-write register**, always needs the user's
  go-ahead, and agents never apply it; the agent writes the migration file and says so.

Then minimize the human column:

1. **Verify that each human item really is human-only.** It may be something an available tool,
   MCP server, already-authenticated CLI, secret manager or subagent can do. If so, re-classify it.
2. **Batch and front-load the human items.** Group them into as few touchpoints as possible and
   schedule them before or in parallel with the agent work, so nothing stalls mid-run.
3. **Make each touchpoint one step.** If an `unblock-you` skill (or a similar hand-off skill) is
   available, apply it. Otherwise apply its rules inline:
   - a direct deep link to the exact screen, never a homepage;
   - any command the human runs, in its own code block;
   - secrets read from the clipboard (for example `"$(pbpaste)"`), never pasted into chat;
   - the exact check the agent runs afterwards to confirm it worked.
4. **Never fake a human step.** If the agent cannot verify something, the plan says so.

## Step 6: Write the plan

The plan starts with the **marker**, so `loop-from-plan` does not have to guess the mode:

```
Execution mode: normal | multi-agent
Tracker: <Linear parent issue>, <GitHub tracking issue owner/repo#N when multi-agent>
Max parallel agents: <N>   (multi-agent only, default 4)
Models: supervisor <model>, implementers <model>, reviewer <model or "none">
```

Then this structure, written to the plan file:

```markdown
# Plan: <short title>

**TL;DR:** <outcome in one line> · Size: <XS–XL, hours> · Units: N · Mode: normal|multi-agent · Human touchpoints: N

## Context
Why this work exists, what prompted it, the intended outcome.

## Diagnosis (only when it is a problem)
The `diagnose` report: root cause with evidence, classification, confidence.

## Inventory (only for a backlog)
Each issue with its verdict and evidence; the cleanup table (what moves to Done, Duplicate or out).

## Scope
In scope / out of scope / assumptions (including every "I don't know").

## Units
One section per unit (template below), then the waves and the conflict hot spots.

## Acceptance criteria (whole plan)
What the user observes when everything is done: observable from the product, not only from
the database.

## Human touchpoints
| # | What | Why it is human-only | When (before/parallel to unit N) | Exact steps | Agent check afterwards |

## Production-write register
| # | Write | Unit | Who approves | Rollback |

## Execution (multi-agent only)
Agent brief, merge gate, who merges, limits, recommended session setup, authorization for the run.

## Risks and rollback

## Readiness checklist
The 17 lines below, each marked pass or fail.

## Linear and GitHub map
Project and milestones, parent issue, sub-issues, plan document, GitHub ledger and links.
```

**Unit template:**
```
### U<n> · <name>   (<size>)   depends_on: [..]   parallel with: [..]
model:         <implementer model>        reviewer: yes | no
journey:       what the user does and sees, from the app, the API or the MCP
acceptance:    observable results ("I open X and see Y", "query Z returns ≥1 row from today")
gate:          the tests and replay or journey cases that must pass
owned files:   paths the agent may edit
shared files:  path -> the single line it may add
contracts:     seam -> writer / reader / shape / latency budget (both sides)
steps:         agent / human, with files, services and tools
prod writes:   entries of the register, or none
blocked on:    the user | another repo | nothing
rollback:      revert the PR; rollback SQL when there is a migration
not in scope:  what looks related but is not this unit
```

Acceptance criteria are mandatory, at plan level and per unit. Write them as things a person
observes or a command returns, never as "implemented" or "works".

**Readiness checklist.** Mark every line pass or fail in the plan. A plan with any fail is a draft:
fix it before `ExitPlanMode`. Lines 9 to 12 apply only to multi-agent plans; mark them "n/a" in a
normal one.

1. Every unit has acceptance criteria that are observable, and a machine-checkable gate. "Tests
   pass" alone is not a criterion.
2. Dependencies form a graph with no cycles; units marked parallel share no owned file.
3. Every unit lists its owned files; overlaps are resolved by order or by the dispatch-line rule.
4. Contracts touched are written for every seam a unit crosses, both sides (writer and reader).
5. No unit is larger than one PR; an L unit says why it cannot split.
6. Every production write is in the register and marked as needing the user's go-ahead.
7. Human work and work in another repo are listed apart and do not block unrelated units.
8. Each unit says how to roll back.
9. The run's state can be rebuilt from the tracker alone: merged PR = done, open PR = in review,
   issue without PR = todo, waiting on the user = blocked.
10. Names are deterministic (`<session branch>-<unit>`), so re-running an iteration is safe and a
    unit already merged is skipped.
11. The merge gate is a literal checklist. It says who merges and which units also need a separate
    reviewer and the user's go-ahead (state semantics, classification, migrations).
12. Limits are set: the concurrency cap, at most two retries per unit before it is blocked, and
    what happens when a unit's premise turns out false (close it with the evidence, do not build it).
13. Every PR is required to carry "contracts touched", the evidence and a "not verified" section;
    silence never means checked.
14. The supervisor and each unit have a model, with the reason.
15. Each unit starts by re-reading its files and re-checking its premise against the default branch.
16. The plan lives where a new session can read it (Linear and the GitHub ledger), not only in a
    local file.
17. There is an out-of-scope list and acceptance criteria for the whole plan.

Present the plan with `ExitPlanMode`.

## Step 7: Agent brief (multi-agent plans)

Every unit uses the same brief, written into the plan:

```
You implement unit U<n> of the plan in <tracker>.
Read the plan and your unit section first. Re-read the files you will touch and re-check that
the premise still holds on the default branch; if it does not, stop and report.
Work in your own worktree on branch <session branch>-u<n>. Edit only the owned files; shared files
get only the single line the unit lists, with the logic in a new module.
Never apply a migration, backfill, flag, secret or deploy, and never send a message on the user's
behalf: write the migration file and say so in the PR.
Run the full test suite and the unit's gate cases; regenerate any generated file the repo checks
in. Walk the unit's journey and paste the evidence.
Open a PR to the default branch: `Closes #<unit issue>` at the top level, contracts touched,
evidence, "not verified". Subscribe to the PR; do not merge it.
Return: the PR number, whether it has a migration, and what needs the user.
```

## Step 8: Create everything (after approval)

Only after the user approves the plan. Search before creating anything (`list_issues`,
`list_projects`, `list_documents`, `gh issue list`) and reuse what already exists instead of
duplicating it.

1. **Backlog cleanup.** Apply the inventory's cleanup table: Done, Duplicate or moved out, each
   with a comment that carries its evidence.
2. **Container.** For an epic or a backlog, use the matching Linear project, or create one with
   `save_project` if none fits, and create one milestone per wave or phase with `save_milestone`.
   For non-epic work, create no new project; use the team's existing project if one applies.
3. **Parent issue.** One issue for the whole plan (`save_issue`), with the TL;DR, the marker, the
   plan-level acceptance criteria and a link to the plan document.
4. **Sub-issues.** One per unit, created with the parent issue set. Each has its acceptance
   criteria, gate, owned files and dependencies, and is assigned to its milestone. Human
   touchpoints become their own sub-issues, assigned to the person who owns them and marked as
   blocking the issues that need them.
5. **Plan document.** Publish the approved plan as a Linear document (`save_document`) in the
   project, or wherever the workspace keeps plan docs, titled `Plan: <title>`. Link it from the
   parent issue, and link the parent issue and the sub-issues from the document.
6. **GitHub ledger.** For a multi-agent plan, or whenever the repo's rules require GitHub
   issues, do two things:
   - Create a GitHub tracking issue that carries the whole plan, starting with the marker.
   - Create one GitHub issue per unit, cross-linked both ways with its Linear sub-issue.

   `loop-from-plan` rebuilds the run's state from these issues and their PRs. Prefer Linear's own
   git branch name for the branches where the repo has no other rule.
7. **Statuses.** Set statuses from the team's real workflow: everything `Todo` or backlog. Nothing
   goes to `In Progress` until execution starts on it.

## Step 9: Hand off to execution

Reply with:
- the Linear parent issue, sub-issues and plan document links, plus the GitHub tracking issue when
  there is one;
- the human touchpoints the user should do now, each ready in one step as in Step 5;
- the next action.

Do not start implementing until the user says so.

**Normal plan.** Offer `loop-from-plan`, telling it to reuse the issues already created.

**Multi-agent plan.** Tell the user to run `/dashone-community:loop-from-plan` on the tracking
issue in a fresh session. It reads `Execution mode: multi-agent` and runs the supervisor loop.
Give the recommended session setup with the reason for each part:
- **Supervisor:** the strongest available model. Its mistakes, such as merging a bad PR, are the
  expensive ones, and its token use is small.
- **Implementers:** a faster model.
- **Reviewer:** a separate reviewer for risky units.
- **Permissions:** production writes stay gated through the permission settings.
- **Timer:** a timer set inside a container dies with it. For a run that must survive a restart,
  use a pending scheduled wake-up.

Also give the **authorization the new session needs in its first message**, because permissions
granted in this session do not carry over: which branches agents may push to, whether the
supervisor may merge when the gate passes, and what it must never do without asking.

## Rules

- Never plan work for something the inventory found already done; plan its closure.
- Never claim a unit is verified when nobody ran the journey; the plan says who walks it.
- Do not create issues, change Linear or GitHub, or write to production before the plan is approved.
- Never infer parallelism that the owned files and dependencies have not proven.

## Failure modes to avoid

- Planning on a model other than Opus 5.5 or Fable, or outside plan mode without strict read-only.
- Planning a fix on an unconfirmed cause.
- Planning a backlog from titles, without reading each issue and checking it against the code.
- Grouping by issue title, which sends two agents to the same file.
- Acceptance criteria that read "implemented" or "tests pass".
- Leaving a migration or a backfill to the agent "if it is safe".
- Keeping the plan only in the planning session or in a local plan file.
- Handing over a plan with a failed checklist line because the rest looked good.
