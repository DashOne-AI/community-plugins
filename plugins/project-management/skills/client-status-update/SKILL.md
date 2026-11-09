---
name: client-status-update
description: Project status from Linear, GitHub and Drive with a gap check on dates, claims and blockers. Two modes - a weekly client update (doc plus short message) and a daily internal status snapshot written to the Project Index (status collection) for an at-a-glance view of what is urgent, waiting, next, missing and who owns it. Use for status updates, daily status, status diario, what is pending, overdue or blocked, and scheduled daily runs. Works with requests in Spanish or English.
---

# Client status update

Goal: a one-page update the client reads in two minutes and acts on without follow-up questions, plus a short message to send it with. The user sends it. Never send anything to the client yourself.

Core rules, apply at every step:
- Tools are truth. Chat claims, memory and old docs are leads to verify, not facts.
- Never invent a date, number, owner, result or client ask. Missing means ask, or leave it out.
- Report change since the last update, not just current state.
- Every slip is owned: what moved, why, the new commitment.
- Client output has no internal codes, links, tools, blame or other clients.

## Step 0. Scope the run

Resolve from the request, memory and the project index. Ask only what is missing, in one message:
- Project and client. Audience: name and role (owner or exec gets outcomes, operator gets more task detail).
- Channel (WhatsApp, email, meeting) and language. Default to the language of past client messages. Spanish means Mexican Spanish with full accents, tildes and inverted punctuation.
- Period: since the last update sent to this client. If unknown, the last 14 days, and say so.

## Step 1. Gather (silent, parallel where possible)

1. Today's date and weekday from the time tool. Never assume it.
2. Project profile: client contacts, sites or domains, repos, owners, charter or contract milestones, client-facing artifacts. Check the project index, memory, and the Linear project description.
3. Linear: milestones (progress, targets), all issues (status, due, labels, assignee, priority), comments and history for date changes in the period, existing project status updates.
4. GitHub: PRs merged and open in the period for mapped repos. Merged and deployed counts as shipped. Open PR counts as in review.
5. Last update sent: past chats, sent email, prior update docs. Extract every promise made (dates, deliverables, asks). This is the commitments ledger.
6. Client-facing artifacts (hub, dashboard, plan, charter): last sync date.
7. Optional, only if connected and verified: real outcome metrics (traffic, leads, AI mentions). Never estimate.

If a source is not connected, name it in the hand-off and continue with what is available. Do not stall.

## Step 2. Reconcile and score

Build an internal gap list, never shown to the client:

- Ledger check: each promise from the last update is kept, slipped, or silently dropped. A silent drop is the worst case and must be addressed.
- Past due with no new date or reason.
- Dates moved without an explanation comment.
- Done in chat but not done in tools (open PRs, in-progress issues). Tools win.
- Work tagged to the wrong site, workstream or domain, or not tagged.
- Misleading progress figures (issue-count percentages). Prefer concrete shipped items.
- Charter or contract checkpoints that are due soon or behind.
- Client-facing artifacts older than the current state, and by how many days.
- Unverifiable names, owners or dates from the user.
- Date math: every weekday and date pair checked against the calendar.

Then set an internal health per workstream and overall: On track, At risk (a committed date at risk but recoverable), Off track (a committed date missed with no new commitment). This drives tone. It is never shown as a label to the client.

## Step 3. Confirm (one batch, max 4)

Ask only what changes the document. Use multiple choice with the recommended option first. Pick from:
1. New committed date for each slipped or at-risk item the client will ask about.
2. Real client asks or blockers (access, content, approvals), each with a due date.
3. What to exclude.
4. Who owns strategy or other updates, and when they will share.

Defaults when unanswered, the user is away, or the user says make the call:
- No confirmed date: keep the plan target and say dates are being set with the owner. Never invent one.
- No confirmed client ask: omit the asks section. Do not write that the client has no blockers unless the user confirmed it.
- Keep unverified items out of the doc.
- List every default used in the hand-off.

## Step 4. Build the client doc

Create a new dated doc each run (title: <Client> Project Update - <Mon DD, YYYY>). Do not overwrite earlier updates. They are the record. Skeleton first, then one section per call. US English unless the client language is different. No em dashes or en dashes. Plain words.

Sections, in order. Hard budget: 350 words, not counting the table.
1. Lead sentence: state of the project and the next goal.
2. Since last update: 3 to 5 shipped items, each with what it does for the client. In-review items are labeled as such.
3. This week: what closes and when, one line on why it matters. Moved dates show old and new, plus a one-clause reason.
4. Roadmap: table of phase, what it includes, target. Only targets from the plan of record or confirmed by the user.
5. What's next and why: the strategy logic in 2 or 3 sentences (for example, pages alone do not bring traffic, so the lead magnet comes first), and who owns it.
6. What we're watching: 2 or 3 items, each with the response. Frame as risks being managed, never as excuses.
7. What we need from you: only confirmed asks, each with a date.
8. Next update: date and channel, only if the user confirmed a cadence.

Leave out: issue codes, PR numbers, tool names, internal links, process talk, stale artifact problems, health labels, team issues, anything excluded. No percentages unless they mean something to the client.

## Step 5. Challenge pass

If you can start a sub-agent, have it read the doc cold as the client, with no other context, and list every question it would ask. Otherwise do it yourself. Fix or cut until:
- The state is clear from the lead sentence alone.
- No line raises a question the doc does not answer.
- No slip appears without a reason and a new commitment.
- Every claim traces to a tool result or the user.
- Nothing contradicts the last update or a client-facing artifact. If an artifact is stale, flag it internally before sending.
- No promise appears that the user did not confirm.
- It is within the word budget.

## Step 6. Companion message

Under 90 words, in the user's voice and the client's language. Apologize for silence only if there was a real gap since the last update. Include the one-line state, this week, who owns the next strategic update, and the doc link. Return it in chat ready to copy. Email: offer a draft, never send without a clear yes. WhatsApp: if no tool can send it, say so once and leave it at copying.

## Step 7. Hand-off and follow-through

Reply short:
- Doc link and the message.
- Assumed or omitted: one line each.
- Sources not reached, if any.
- Internal follow-ups: stale client artifacts, slipped items with no new date, missing labels, ledger breaks. Offer to fix them (reschedule, relabel, add a reason comment, refresh the hub).
- Offer to log an internal Linear project status update with the health rating and the commitments made in this update, so the next run has a ledger.

Do not repeat the doc content in chat.


## Daily mode (internal status in the Project Index)

Use when the request or a scheduled run asks for the daily status. It is internal: never send it to the client and never create a Claude Doc for it. The output is one record per day in the Project Index database, rendered by the index page as the "Daily status" view (tabs by day, live Linear layer).

Run unattended. Ask nothing; apply the Step 3 defaults and list them in `notes`.

1. Resolve: today's date in the project time zone (default: the user's time zone), the Project Index URL (as in `project-index-manager` step 1), the Linear project, and the people map (the index page's `BRAND.status.people`; Linear users map to people through each person's `match` list).
2. Gather as in Step 1, items 1, 3, 4 and 5, plus the latest record in the index `status` collection (yesterday's snapshot). The commitments ledger comes from the last client update.
3. Reconcile as in Step 2 and set health with the same rules.
4. Write the record: `write_db` `set`, collection `status`, `doc_id` = `YYYY-MM-DD`. A rerun the same day replaces that day's record (pass `if_version`). Never edit past days.
5. Nightly run (the default schedule, 21:59 in the project time zone): the record is for the next day. Use tomorrow as `date` and `doc_id`, compute lateness against tomorrow, and leave out issues already Done or Canceled. The index page crosses yesterday's record with live Linear during the day, so closed items show as done until this run replaces it. If a run is missed, the page flags the record as out of date and tells the user to wait for the next run or to run the scheduled task now.

Record fields (US English, plain words, no em or en dashes, typographic apostrophes):
- `date`, `generated_at` (UTC ISO), `sources` (list of what was read), `health` (`on_track` | `at_risk` | `off_track`), `health_reason` (one sentence), `headline` (35 words max: the state and the one thing to do today).
- `counts`: `open`, `overdue`, `unassigned`, `changed` (since yesterday).
- Lists of items, each `{text, why?, owner, due?, late_days?, refs: [{id, url}]}`. `text` is 12 words max and starts with a verb or the deliverable; `why` is 25 words max; `owner` is a person from the map or "Unassigned"; `late_days` only when past due.
  - `urgent` (3 max): client commitments due within one business day or already past, work that blocks a client commitment, and client inputs received but not yet acted on. Order by impact.
    Each urgent item also carries `actions` (2 to 3), the buttons the user presses to start moving:
    - `{kind: "chat", label, prompt}`, always first: opens a new Claude chat with `prompt` filled in. The prompt is self-contained (project, what was promised and when, the Linear issue links, what to produce), asks to read the sources before acting, forbids inventing dates or facts, and says not to send anything or change Linear without asking. 900 characters max.
    - `{kind: "draft", label, prompt}`, when the next step is a message: the page asks Claude for the draft in place. Name the sender, the recipient, the channel, a word limit and the facts; use placeholders like [new date] for anything unconfirmed; end with "Return only the message."
    - `{kind: "link", label, url}`: the Linear issue or project to open.
    Labels are 2 to 5 words and start with a verb ("Plan the reset in chat", "Draft the note to the client", "Open ABC-29").
  - Any item in `urgent`, `next` or `decisions` whose Linear issues have no assignee carries `assign`: up to 3 `{id, title}` (title 6 words max). The page shows "no owner · Assign to" with one button per assignable person, which sets the assignee in Linear through the viewer's connector, with undo. Only list issues that are unassigned in Linear at run time and that someone should pick up now.
  - `waiting` (5 max): what others owe, with days late. Client inputs go to the client contact even when unassigned in Linear; record that in `owner_overrides`.
  - `next` (8 max): due in the next 5 business days, by owner.
  - `gaps` (6 max): overdue with no new date, no owner, milestone dates that contradict the client commitment, no due date on high priority work, stale tracker updates, unlabeled work. Counts plus a few refs, not full lists.
  - `decisions` (3 max): questions only the user can answer, each with an owner.
- `milestones`: `{name, progress (0 to 100 from Linear), target (Linear), committed (date promised to the client, when different), health: on_track | at_risk | late | not_started, note?}`.
- `changes`: `{closed, opened, moved, updated}` against yesterday's record: issues completed, created or updated in the period, and due dates that differ from yesterday's `due_index`.
- `owner_overrides`: `{issue_id: person}` for items whose real owner is not the Linear assignee. The page uses it for the live layer.
- `due_index`: `{issue_id: due_date}` for every open issue with a due date, so the next run can detect moved dates.
- `notes` (4 max): defaults used, sources not reached, assumptions.

Every item traces to a tool result or the user. If a fact cannot be verified, leave it out and say so in `notes`. Never estimate numbers.

Hand-off in chat, two lines max: health, the top urgent item, and the index link. If the run is scheduled, write nothing else.

## Style

Minimal words in chat. No preambles. Links on any reference to another document or tool in internal output. Client output carries no internal links.