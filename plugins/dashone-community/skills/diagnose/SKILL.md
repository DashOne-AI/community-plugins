---
name: diagnose
description: Diagnose a production issue, bug, alert or incident and deliver a complete analysis (root cause with evidence, impact, severity, urgency, priority, effort, risk, whether it qualifies as a hotfix, and a fix plan) in plan mode with Opus 5.5. Asks for the needed context before investigating. Works with requests in Spanish or English. Use when the user says "/diagnose", "diagnose this", "diagnostica esto", "analyze this bug/alert/incident", "analiza este bug/alerta/incidente", "how bad is this", "qué tan grave es esto", "triage this issue", "triage de este issue", or pastes an alert, a stack trace or a GitHub/Linear issue asking for analysis. Do NOT use to implement the fix (that comes later, once the user approves the plan).
---

# diagnose

Read-only diagnosis → standard analysis → approvable plan. No code or data is touched until the
user approves the plan.

All output goes in the language the user used. If that is Spanish, use neutral Latin American
Spanish (see your global CLAUDE.md).

## Step 0: Opus 5.5 + plan mode (before anything else)

A skill's frontmatter cannot change the model or the permission mode; this is done with tools:

1. **Model.** If the current model is not Opus 5.5 (`claude-opus-5-5`):
   - If `mcp__ccd_session_mgmt__set_session_model` exists (load it with ToolSearch), use it to
     set `claude-opus-5-5` for this session and, if `set_session_effort` exists, set effort to
     `xhigh` (per your team's agent rules: hard debugging → Opus, `xhigh`).
   - If it does not exist, tell the user in one line: "Switch the model to Opus 5.5
     (`/model opus`) and run `/diagnose` again" and stop. Do not do the diagnosis with another
     model.
   - If it is already Opus 5.5, say nothing and continue.
2. **Plan mode.** Call `EnterPlanMode` (load it with ToolSearch if it is deferred). If the user
   rejects it, continue anyway but in strict read-only mode: no Edit/Write, no commands that
   write, migrate, deploy, retry runs or spend money.

## Step 1: Ask for context (always, before investigating)

First extract what already comes in the message, issue, alert or args (`$ARGUMENTS`). Then ask
ONLY for what is missing, in a single `AskUserQuestion` (max. 4 questions) or a short text list if
the answers are free-form. The minimum needed to diagnose well:

1. **Exact symptom:** what was expected vs. what happened. Error message, screenshot, run/
   request/trace ID, URL.
2. **Where and since when:** repo/service, environment (prod/staging/local), first time it was
   seen, whether it is constant or intermittent, and what changed around that date (deploy,
   migration, config, secret, provider).
3. **Who is affected:** which client(s)/users, how many, whether there is a workaround today.
4. **What has already been tried** and what was ruled out.

Do not ask for what you can verify yourself (logs, `gh`, `fly`, Supabase, Doppler, git): you
investigate that in step 2. If the user says "I don't know", continue and mark it as an
assumption.

## Step 2: Investigation (read-only, with evidence)

Rules from the global CLAUDE.md section "Debugging: identify the REAL process/state before
fixing":

- **Sync before reading:** `git fetch` and read against `origin/<branch>` (`git show`) if the
  local checkout is behind. Check `git submodule status`.
- **Identify which process, which build and which config produced the symptom**, with real
  observability (logs, run/event tables, `fly releases`, `/version`, deploy vs. last commit). Not
  by deduction.
- **Frontend "nothing happens" bug:** reproduce it live (browser pane or Claude in Chrome),
  comparing expected vs. actual network/console.
- For search fan-out use `Explore` subagents or an equivalent search subagent; for long logs use
  a log-reading subagent. Launch independent work in parallel.
- Each hypothesis ends up **confirmed** (with evidence: file:line, query, log), **ruled out**
  (with why) or **open** (what is missing to close it). Never present a hypothesis as the cause.
- Reads from prod yes; writes, retries, re-runs or any spend, no.

## Step 3: Standard analysis

No team guideline defines a severity scale yet; this is the skill's own. If the repo has its own
standard (`AGENTS.md`/`CLAUDE.md`/`docs/`), use it and say so.

### Severity (how serious the damage is)

| Level | Criteria |
|---|---|
| **S1 Critical** | Prod down or a core flow impossible for clients; data/credentials exposed; data loss or corruption; an incorrect number that a client will use to decide (price, balance, score, invoice); runaway spend. |
| **S2 High** | Important flow degraded for several clients or one key client, with no reasonable workaround; partial results presented as complete. |
| **S3 Medium** | Affects few, with a workaround; visible error but not misleading; secondary functionality. |
| **S4 Low** | Cosmetic, copy, log/alert noise, technical debt with no user effect. |

### Impact (scope) and urgency

- **Impact:** who (named clients, internal users, nobody yet), how many, which flow, money (API
  costs, billing), data (integrity, privacy), reputation/contractual. Quantify with real data when
  possible (rows, runs, requests, USD); otherwise give a range + assumption.
- **Urgency:** does it get worse over time? (data being corrupted, spend accumulating, a weekly
  run that gets lost, a demo/delivery with a date).

### Priority = severity × urgency

| | High urgency | Medium urgency | Low urgency |
|---|---|---|---|
| **S1** | P0 (now) | P0 | P1 |
| **S2** | P1 (today/tomorrow) | P1 | P2 |
| **S3** | P2 (this cycle) | P2 | P3 |
| **S4** | P3 | P3 | P3 (backlog) |

### Is it a hotfix? (rule from your team's guidelines, all three or it is not one)

1. Prod is broken **now**: someone cannot complete a flow, sees an incorrect number they are going
   to use, or data/credentials are exposed.
2. It cannot wait for the next `staging → main` cycle: there is no workaround.
3. The fix is small and reversible: few files, no destructive migration, no new dependency.

It is not a hotfix: something new, cosmetic, a bug with a workaround, a bug that went weeks
without being reported, "I need it for tomorrow's meeting" (that is priority). Answer Yes/No with
each condition evaluated.

### Effort

Size + concrete hours for whoever executes it (Claude or a person), including tests, migration and
verification in the real environment: **XS** < 1 h · **S** 1 to 4 h · **M** ½ to 2 days ·
**L** 2 to 5 days · **XL** > 1 week (split into slices). Say what pushes the estimate up (e.g.
"if there are no tests covering X, it goes up to M").

### Fix risk and confidence

- **Fix risk:** what it can break, whether it touches migrations/a shared DB, whether it is
  reversible, rollback plan.
- **Diagnosis confidence:** 1 to 10. Below 8, say what evidence is missing to raise it and whether
  it is worth getting it before fixing.

## Step 4: Deliver as a plan (ExitPlanMode)

The plan presented with `ExitPlanMode` is the complete report, in this order:

```markdown
# Diagnosis: <short title>

**TL;DR:** <cause in one line> · **S?/P?** · Hotfix: Yes/No · Effort: <size, hours> · Confidence: N/10

## Symptom
Expected vs. observed, environment, since when, how to reproduce it.

## Root cause
Which process/build/config produces it and why, with evidence (file:line, logs, queries).
Causal chain: trigger → mechanism → symptom.

## Hypotheses
| Hypothesis | Status (confirmed/ruled out/open) | Evidence |

## Impact
Who, how many, which flow, money, data, since when. Real numbers or range + assumption.

## Classification
Severity (S1 to S4) + why · Urgency + why · Priority · Hotfix (3 conditions evaluated)

## Immediate mitigation (if applicable)
What reduces the damage today without the full fix (workaround, pausing a job, notifying a client).

## Fix plan
1 to 5 numbered steps: files to touch, tests to add (one that fails before the fix), migration if
applicable, branch and git flow (hotfix/* or fix/* → staging), verification in the real environment.

## Effort and risk
Size + hours, what changes it, fix risk, rollback.

## Open questions / assumptions
What could not be verified and how to close it.

## Next step
A single concrete action.
```

After the user approves: if the diagnosis came from a GitHub issue, propose adding the analysis as
a comment and marking it "In Progress" in its Project (rules from the global CLAUDE.md); if there
is no issue, offer to create one and add it to the Project. Do not implement anything the user has
not approved.
