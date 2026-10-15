---
name: loop-instructions-builder
description: Turns an approved plan (or any task to run autonomously or in a loop) into a tailored execution-authorization prompt, already checked against Claude Code's real hard limits instead of discovering them mid-run. Analyzes the specific plan for hard-blocked steps (security/system settings, credentials, financial actions, destructive ops), carves them out to the user, preserves sequencing safeguards, and adds a stop-and-flag protocol for anything that blocks mid-run. Use BEFORE telling Claude "implement everything end to end, don't ask me anything," before /loop, or whenever the user is about to write a broad autonomy prompt for infra, deployments, integrations, auth, migrations, or anything touching dashboards or credentials. Trigger on requests like "turn this plan into an autonomous prompt" or "run this in a loop without asking me anything", even if the user does not name this skill or already wrote their own blanket-authorization prompt. Works with requests in Spanish or English.
---

# Loop Instructions Builder

## Why this exists

Users writing an "execute autonomously, I authorize everything" prompt tend to write it optimistically: they picture the happy path, not the specific step in their own plan that Claude is structurally unable to do alone (or a step that must not run before another one finishes). Discovering that mid-execution wastes the whole point of running autonomously, the run stalls exactly when no one's watching.

The fix isn't a nicer-sounding authorization prompt. It's checking the *specific* plan against Claude Code's actual operating limits **before** the run starts, so the authorization prompt is accurate instead of aspirational. A blanket "I authorize everything" cannot make a hard-blocked action doable, it can only hide the fact that it's blocked until Claude hits it.

## What this skill produces

Not a filled-in template. A prompt built fresh from a real analysis of the plan/task in front of you right now. Reusing wording from an old run because it "sounds right" defeats the purpose, the whole value is in re-deriving which steps are actually blocked *this time*.

## Step 1: Find the real plan

Look for the actual thing being authorized, in this order:
1. An active plan-mode file (check for a recent file under `~/.claude/plans/` or wherever the current session's plan was written).
2. A plan the user pastes or points you to.
3. If neither exists, ask the user to describe the task in enough detail to do step 2 for real, a one-line task description isn't enough to check for hidden blockers.

Read the plan closely enough to list its concrete steps, not just its summary. Blockers hide in steps, not in headlines.

## Step 2: Check every step against Claude Code's actual limits

This is the part that must not be skipped or done from memory, go through each step and classify it. These categories come from Claude Code's own operating rules and hold regardless of what the user authorizes:

**Credentials and auth: check the secret manager and connectors before flagging as blocked:**
For any step that needs a password, API key, token, or authentication, first check whether the user's secret manager (for example Doppler) or an authorized connector (for example Composio) already holds the needed secret or an authorized connection for it. These are pre-authorized secret and token stores, not a workaround channel. If one of them has it, use it through that channel and the step is NOT hard-blocked. Only fall into the hard-blocked bucket below if you have checked and confirmed the secret or connection genuinely does not exist in any of them. In that case, say so plainly to the user ("checked the secret manager and connectors, neither has X, this one is yours") rather than assuming it is missing.

**Hard-blocked, always the user's, no matter the authorization:**
- Entering passwords, API keys, tokens, or financial/ID credentials into any field, when not already available through the secret manager or connectors (see above)
- Creating accounts, or authenticating on the user's behalf, when no existing connector or secret-manager entry covers it
- Permanently deleting data
- Executing a financial trade or money transfer
- Giving personalized financial/investment advice
- **Modifying system or security settings**: this one catches more than it sounds like: auth-provider setup (OAuth/SSO/third-party auth registration), IAM/permission changes, anything that changes who is trusted to authenticate as whom
- Bypassing CAPTCHAs or bot-detection
- Downloading/running files from untrusted sources

The channel doesn't matter. Whether it's attempted via CLI, an API call, or driving the user's own logged-in browser, if Claude performs the action, it's still blocked. Say this explicitly in the output, users reasonably assume "but I gave you my browser" changes the answer. It doesn't.

**Needs a specific ask when it comes up, not something to pre-clear in bulk:**
- Downloading a file, sending a message on the user's behalf, publishing or modifying public content, purchasing with a stored payment method, accepting terms/OAuth grants, changing account settings, creating standing/persistent config (webhooks, forwarding rules, cron), submitting a form, any irreversible one-click action.

These aren't forbidden, they just can't be authorized in advance as a category. A prompt that says "I authorize everything" is trying to pre-clear this bucket wholesale; it can't. Say so plainly in the output so the user isn't surprised when Claude still asks about one of these mid-run.

**Regular:** everything else. This is most of most plans, code changes, reading data, running tests, applying migrations the user already has access to, git/PR workflows, calling already-authorized CLIs and MCPs.

For each hard-blocked step found, also check whether anything downstream in the plan *depends* on it being done first. If step 4 breaks visibly (not just "in theory") unless step 1 is verified complete, that's a real sequencing constraint, name the concrete failure mode, not just "do things in order." If you can't articulate what actually breaks and how, it's not a real sequencing constraint, don't invent one.

**If a plan has zero hard-blocked steps, say so directly**: produce a short prompt confirming full autonomy instead of forcing a blockers section that doesn't apply. Manufacturing a caveat where none exists is as much a failure here as missing a real one.

## Step 3: Write the prompt

Same language as the plan/conversation. Fixed shape, but every line's content is derived from step 2, not copied from a prior run:

1. **What's authorized to run without asking again**: an actual list of what the plan calls for, never the word "everything."
2. **The hard-blocked carve-out** (only if step 2 found one), name the exact step, why it's the user's alone, and the explicit "don't attempt this via any channel, including my own logged-in session, even though I've authorized the rest" framing. For credential/auth steps specifically, state that the secret manager and connectors were checked first and confirmed not to have the secret or connection. Do not carve out a credential step without naming that check.
3. **Sequencing constraint** (only if real), what must be verified before what, and the concrete thing that breaks if skipped.
4. **The stop-and-flag block, reusable verbatim:**
   ```
   ### BLOCKED: I need you to do this
   - What exactly is missing
   - Why I cannot do it myself
   - Exact steps for you to resolve it
   - What I will keep doing in parallel in the meantime
   ```
   (Translate the heading/labels to the plan's language.) The point of this block is that it's scoped to the one blocked sub-step, Claude keeps working on everything else in parallel rather than treating one blocker as a reason to stop entirely.
5. **Testing-rigor instruction**: real end-to-end verification, not a single happy-path API call. Push specifically for edge cases the plan's own verification section implies. If achieving E2E would otherwise require impersonating the user (their login, their session), redirect it: use an API-issued token/test account instead, and never fake a check that couldn't actually be run.
6. **Anti-blanket clause**: the authorization in this prompt covers exactly what's listed in (1); anything new that comes up mid-run in the "needs a specific ask" bucket still gets asked about specifically, on the spot.
7. **Closing summary request**: ask for: what got done and verified, what's sitting in the BLOCKED section waiting on the user, and the exact rollback if something needs to be undone.

Sections 2 (hard-blocked carve-out) and 3 (sequencing constraint) are conditional, include them only when step 2 actually found something concrete; don't pad with a hollow paragraph just to hit the shape. Sections 1, 4, 5, 6, and 7 always belong in the output, even for a plan with zero hard blockers: the stop-and-flag block (4) is a forward safety net for something the plan didn't foresee, not only a home for known blockers, and 1/5/6/7 apply to any autonomous run regardless of what step 2 turned up.

## Step 4: Deliver it

One markdown code block, ready to copy/paste or send as the next message. No extra commentary buried inside the block itself, anything you want to tell the user about your reasoning goes outside the code block, not inside it.
