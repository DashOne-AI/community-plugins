---
name: linear-plan
description: Plan a piece of work end to end and leave it ready to execute in Linear — on Opus 5.5 or Fable only, after 3 to 10 gap-closing questions, running `diagnose` first when the request is a problem, splitting epic-size work into phases, writing acceptance criteria, creating the Linear issue and sub-issues, and publishing the plan as a Linear document. The execution strategy minimizes what the human implementer has to do. Works in Spanish or English. Use when the user says "/linear-plan", "planea esto", "arma el plan", "haz un plan para…", "planéalo en Linear", "plan this", "make a plan for…", "plan this in Linear", or asks to turn an idea, feature, bug or incident into an executable plan with tickets. Do NOT use to implement the plan (hand that to execution, e.g. `loop-from-plan`, once the user approves).
---

# linear-plan

Idea or problem → closed gaps → phased plan with acceptance criteria → Linear issue, sub-issues
and plan document → ready to execute with as little human work as possible.

All output goes in the language the user used. If that is Spanish, use neutral Latin American
Spanish (see your global CLAUDE.md).

## Step 0: Opus 5.5 or Fable + plan mode (before anything else)

A skill's frontmatter cannot change the model or the permission mode; do it with tools.

1. **Model gate.** Allowed: Opus 5.5 (`claude-opus-5-5`) or Fable (`claude-fable-5-*`). Check
   which model you are running as (your system prompt, or `get_session` with `self` if the
   session tools exist).
   - Allowed model: say nothing and continue.
   - Otherwise, if `mcp__ccd_session_mgmt__set_session_model` exists (load it with ToolSearch),
     set `claude-opus-5-5` for this session and, if `set_session_effort` exists, effort `high`
     or above. Continue only once the switch took effect.
   - If no tool can switch it, tell the user in one line: "Switch to Opus 5.5 (`/model opus`) or
     Fable and run `/linear-plan` again" and stop. Do not plan with another model.
2. **Plan mode.** Call `EnterPlanMode` (load it with ToolSearch if it is deferred). If the user
   rejects it, continue in strict read-only mode until the plan is approved: no Edit/Write
   outside the plan file, no commands that write, deploy, migrate or spend, no Linear/GitHub
   writes.

## Step 1: Problem or build?

Classify the request:

- **Problem** — something is broken, wrong, slow, failing or "doesn't work": a bug, alert,
  incident, regression, data that looks off.
- **Build/change** — a feature, improvement, migration, refactor or new project.

If it is a **problem**, run the `diagnose` skill first (`diagnose`, or
`dashone-community:diagnose` when installed as a plugin) through its Steps 1 to 3: context,
read-only investigation with evidence, standard analysis. Do **not** end at its Step 4
`ExitPlanMode`: fold its report into this plan as the **Diagnosis** section and carry its
priority, effort and fix plan forward as the input for Step 4 below. Do not re-ask questions
`diagnose` already asked. If `diagnose` is not available, apply its rules inline: identify the
real process, build and config that produced the symptom with real observability, and mark
every hypothesis as confirmed, ruled out or open. Never plan a fix on an unconfirmed cause.

## Step 2: Find the Linear destination

Infer, do not ask first:

1. The repo: `CLAUDE.md`/`AGENTS.md`/`README.md` mentions of a Linear team or project, the git
   remote, and existing Linear issues that reference the repo (`list_issues` with a keyword
   query).
2. The client or project context the session is in (a client-context skill if one is available,
   the working directory, the client named in the request).
3. Existing Linear projects that match (`list_projects`), so the work lands in one that already
   exists instead of a new one.

If team or project is still ambiguous, it becomes one of the Step 3 questions. Statuses and
labels are per team: read them with `list_issue_statuses` / `list_issue_labels` before using
them.

## Step 3: Close gaps — at least 3 questions, at most 10

Always ask at least 3 questions, and stop at 10. Use `AskUserQuestion` (up to 4 per call, so
up to 3 rounds), or a short text list when the answers are free-form. Each question must stand
alone and name what it is about.

- Never ask what you can verify yourself (code, logs, git, Linear, docs, DB reads). Verify it,
  then ask about what is left.
- Prioritize the gaps that change the plan the most:
  1. **Outcome and done**: what the user observes when it is finished.
  2. **Scope**: what is explicitly out of scope.
  3. **Constraints**: deadline, budget, stack, environments, things that must not change.
  4. **Human touchpoints**: which accesses, accounts, approvals and secrets exist, and who holds
     them (feeds Step 5).
  5. **Linear destination**, if Step 2 did not resolve it.
  6. **Rollout**: staging first, feature flag, who must be told.
- Stop asking once the answers stop changing the plan, but never before 3.
- When the user answers "I don't know", continue and record it as an assumption in the plan.

## Step 4: Size and phases

Estimate the effort with the `diagnose` scale: **XS** < 1 h · **S** 1 to 4 h · **M** ½ to 2
days · **L** 2 to 5 days · **XL** > 1 week.

The work is **epic size** when it is L or XL, spans more than one repo or service, or needs
more than one independently shippable change. Epic-size work is split into **phases**:

- Each phase ships and is verified on its own, in the order that removes the most risk first,
  with its own acceptance criteria.
- A phase is something a person would review and merge as one PR or one small set of PRs.
- Each phase states what it depends on. Phases with no dependency between them are marked as
  parallelizable.

Non-epic work is a single phase.

## Step 5: Execution strategy — as little human as possible

Default to the agent doing the work. For every step in the plan, classify it:

- **Agent** — code, tests, reads, migrations it can apply, PRs, Linear and GitHub updates,
  verification with tools it has. Most steps.
- **Human, hard-blocked** — things the agent cannot do whatever it is authorized to do:
  entering passwords or secrets, OAuth/consent screens, 2FA/OTP codes, pairing a device (a QR
  scan), payments, account-owner approvals, security or system settings.
- **Human, by policy** — things the repo rules reserve for a person (for example a merge that
  needs a human review, a production deploy, a message to a client).

Then minimize the human column:

1. **Verify each human item really is human-only** — not something an available tool, MCP
   server, CLI already authenticated, secret manager or subagent could do. Re-classify it if it
   is not.
2. **Batch and front-load** the human items: group them into as few touchpoints as possible and
   schedule them before or in parallel with the agent work, so nothing stalls mid-run.
3. **Make each touchpoint one-step**: if an `unblock-you` skill (or a similar hand-off skill) is
   available, apply it; otherwise apply its rules inline — a direct deep link to the exact
   screen (never a homepage), any command the human runs in its own code block, secrets read
   from the clipboard (for example `"$(pbpaste)"`) instead of pasted into chat, and the exact
   check the agent runs afterwards to confirm it worked.
4. **Never fake a human step.** If the agent cannot verify something, the plan says so.

## Step 6: Write the plan and get approval

Write the plan file with this structure and present it with `ExitPlanMode`:

```markdown
# Plan: <short title>

**TL;DR:** <outcome in one line> · Size: <XS–XL, hours> · Phases: N · Human touchpoints: N

## Context
Why this work exists, what prompted it, the intended outcome.

## Diagnosis (only when it is a problem)
The `diagnose` report: root cause with evidence, classification, confidence.

## Scope
In scope / out of scope / assumptions (including every "I don't know").

## Phases
### Phase 1 — <name>   (depends on: — · parallel with: —)
- Goal
- Steps (agent / human), with the files, services and tools involved
- Acceptance criteria (observable, verifiable)
- Verification: how it is checked end to end, in the real environment

## Acceptance criteria (whole plan)
What the user observes when everything is done. Observable from the product, not only from
the database.

## Human touchpoints
| # | What | Why it is human-only | When (before/parallel to phase N) | Exact steps | Agent check afterwards |

## Risks and rollback

## Linear map
Project/milestones, parent issue and sub-issues to be created, plan document, GitHub links.
```

Acceptance criteria are mandatory, at plan level and per phase. Write them as things a person
observes or a command returns ("I open X and see Y", "query Z returns ≥1 row from today"), never
as "implemented" or "works".

## Step 7: Create everything in Linear (after approval)

Only after the user approves the plan. Search before creating anything (`list_issues`,
`list_projects`, `list_documents`), and reuse what already exists instead of duplicating it.

1. **Container**
   - Epic: use the matching Linear project, or create one with `save_project` if none fits.
     Create one milestone per phase with `save_milestone`.
   - Non-epic: no new project. Use the team's existing project if one applies.
2. **Parent issue**: one issue for the whole plan (`save_issue`), with the TL;DR, the plan-level
   acceptance criteria and a link to the plan document.
3. **Sub-issues**: one per phase (epic) or per step (non-epic), created with the parent issue
   set, each with its own acceptance criteria and verification, assigned to its milestone. Human
   touchpoints become their own sub-issues assigned to the person who owns them, marked as
   blocking the issues that need them.
4. **Plan document**: publish the approved plan as a Linear document (`save_document`) in the
   project (or attached where the workspace keeps plan docs), titled `Plan: <title>`. Link it
   from the parent issue, and link the parent issue and sub-issues from the document.
5. **GitHub**: if the repo's rules require GitHub issues, create them too and cross-link both
   ways (Linear issue ↔ GitHub issue URL), so the PR can reference both. Prefer Linear's own
   git branch name for the branches.
6. Set statuses from the team's real workflow: everything `Todo`/backlog. Nothing goes to
   `In Progress` until execution starts on it.

## Step 8: Hand off to execution

Reply with: the Linear parent issue, sub-issues and plan document links, the human touchpoints
the user should do now (each ready in one step, per Step 5), and the next action. If the user
wants autonomous execution, offer `loop-from-plan` and tell it to reuse the Linear (and GitHub)
issues already created instead of creating new ones. Do not start implementing until the user
says so.
