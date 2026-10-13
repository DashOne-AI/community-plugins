---
name: landing-page-conversion-analysis
description: "Audit a landing page for conversion effectiveness by walking it as real buyer personas, operating every CTA and form end-to-end, and delivering a prioritized findings document. Use for \"review my landing page\", \"why isn't this converting\", or any conversion/funnel audit."
---

# Landing Page Conversion Analysis

Audit a landing page the way a real buyer experiences it, then deliver a document
of findings and improvements ranked by importance and weight.

The core discipline: **do not review the page by reading it. Operate it.** Most
conversion-killing defects, dead buttons, broken anchors, forms that accept
garbage, flows that dead-end, layouts that collapse on mobile, are invisible to
a read-through and obvious within one minute of actually clicking.

## Scope check (before starting)

Ask only what you cannot infer, and ask it once:

1. **Which personas?** Who is this page trying to convert? If the page itself
   segments (industry tabs, use-case switchers, audience chips), those segments
   are the personas. If the user names one, use it. Default to two contrasting
   ones: the segment the page defaults to, and one it clearly targets but does
   not default to.
2. **Anything off-limits?** Confirm whether submitting forms or triggering
   backend actions is acceptable. Never enter real personal data, real payment
 details, or credentials, use an obviously fictional test business throughout,
   and stop before any irreversible action (purchase, real account creation,
   sending a message to a third party).

If the session is unattended, pick the most reasonable personas, state the
assumption at the top of the deliverable, and proceed.

## Method

### 1. Persona walkthrough

For each persona, traverse the page start to finish and record a **timestamped
journey**: at each moment, what they see, what they think, and whether the page
gains or loses them. Timestamps matter when the page contains timed content
(auto-playing demos, animated conversations, carousels), a persuasive message
that lands at second 45 of a 50-second animation reaches almost nobody.

For each moment capture:

- **What they see**: the actual copy or element, quoted
- **What they think**: the buyer's internal reaction, in their voice
- **Verdict**: wins them / opens a question / loses them

The personas' domain matters. A doctor thinks about liability and patient data;
an accountant thinks about confidentiality, credentials, and whether you invoice.
Ground each persona's objections in the real constraints of their profession,
not generic "is this trustworthy" hand-waving.

### 2. Responsive check

Test at desktop width and at real mobile width (~400-500px CSS). Look for:
navigation overlap, CTAs pushed outside the viewport, horizontal overflow, hero
media clipped, chip/tab rows collapsing into tall columns that push the primary
CTA below the fold, and text that clips instead of wrapping.

Weight mobile findings by the product's actual traffic mix. For a
mobile-native product (messaging, consumer apps, local services), a broken
mobile layout is a P0 regardless of how good the desktop page is.

### 3. CTA inventory: click every one

Build a table of every button and link that asks for an action. For each:
label, location, visual treatment, and **verified status** from an actual click.

Statuses: `OK` · `BROKEN` (wrong destination) · `DEAD` (no response) ·
`INCONSISTENT` (works unreliably) · `UNVERIFIED` (say why).

Then collapse the labels into **intents** (try / buy / get a quote / talk to
someone) and count how many distinct labels and color treatments serve each one.
A single intent wearing eight labels and five colors means the eye can never
learn what anything means. Also check: is there more than one primary button
competing per section?

### 4. Form and wizard walkthrough: operate end to end

Run every multi-step flow to completion with a fictional test entity. Per step
record what it asks, how many decisions it demands, and a **friction rating**
(low / medium / high / blocking).

Probe deliberately:

- **Required fields**: is the requirement real? Enter obviously invalid input
  ("none", "n/a", nonsense) and see whether it passes. A required field that
 validates nothing is the worst of both worlds, it repels people who read the
  asterisk and protects no one who doesn't.
- **Confirmation screens**: do they claim results the system cannot have? A
  screen that reports it "scanned your site and learned your prices" after being
  given a non-URL is trust-destroying theater. Check whether an instant result
  is presented as if work occurred.
- **Decision load**: count the choices per step. Compare against any promise the
  page makes about how long it takes.
- **Exits**: does every step have a working way forward? Does the final step
  actually convert, or dead-end? Does the progress indicator match the real
  number of steps?
- **Scroll and focus traps**: does an inner scrollable container capture the
  wheel? Does completing the flow leave the page in a broken state? Reload and
  check whether the broken state persists.

### 5. Internal coherence check

Cross-read the page against itself. Recurring, high-value contradictions:

- **Demo vs. plan**: does the showcased demo perform capabilities that the
  highlighted plan does not include? This is the most expensive silent defect on
  a pricing page: it converts today and disappoints on delivery.
- **Screen vs. screen**: does one step assert something a later step denies?
- **Promise vs. mechanic**: "takes 2 minutes" against a "1 of 6" counter.
- **Naming**: how many names does the product have for the same thing?
- **Pricing units**: is the billed unit (a "conversation", a "seat", a "credit")
  ever defined? A buyer who does arithmetic will not sign without it.

### 6. Unanswered objections

List the questions each persona asks before giving up contact details, and mark
each `Answered` / `Partial` / `Not answered`, with where it currently lives.
Anything buried in a collapsed accordion counts as Partial at best.

Pay particular attention to objections **the page's own demo creates**. If the
demo shows a document being uploaded, a price being quoted, a calendar being
written to, or credentials being requested, each of those raises a question the
page is then obligated to close.

### 7. Unused arguments

Look for the strongest case the page already contains but never makes, most
often a return-on-investment calculation whose inputs are sitting in the page's
own demo copy or pricing table. Also check for missing urgency the buyer's
calendar supplies (a seasonal peak, a deadline, a renewal cycle) and for absent
social proof in markets that buy on peer referral.

### 8. Score

Rate each dimension 0-10 and give a one-line verdict for the whole page:

- Clarity of the promise
- Segment relevance (does the target buyer land in their own case?)
- Credibility and proof
- Objection handling
- Friction to first action
- Pricing clarity
- Demo-to-offer coherence
- Mobile execution

When auditing multiple personas, score each separately, the gap between them is
itself a finding.

## Prioritization

Every finding gets a **tier** and a **weight**. Never assign time estimates,
calendar dates, or effort in days, importance and weight only.

**Tiers**

| Tier | Meaning |
|---|---|
| P0 | Breaks conversion outright. Dead CTAs, dead-end flows, broken layout on the primary device. No other improvement pays off while these stand. |
| P1 | Structural leak. The visitor arrives, engages, and is lost by design, wrong default, forced field, self-contradiction, buried proof. |
| P2 | Unanswered objection. The buyer wants to say yes and cannot get a question closed. |
| P3 | Clarity and hierarchy. Individually small; together they change how the page reads. |

**Weight (1-10)**: assign from three factors, and state the reasoning in one
clause so the number is auditable:

- **Reach**: what share of visitors hit this? A defect on the default state or
  the primary device outweighs one behind a tab nobody opens.
- **Depth**: how far into the funnel? A defect at the moment of conversion
  destroys all upstream work; a defect in the hero costs one visitor.
- **Certainty**: verified by a click or a screenshot, or inferred? Verified
  findings outrank plausible ones.

Order the backlog by tier, then by weight inside each tier.

## Deliverable

Produce a **document**, published so it survives the conversation and can be
shared with whoever will do the work. Structure:

1. **Verdict**: overall score and a one-sentence judgment that names the real
   pattern, not a platitude.
2. **Top five**: the findings that move the needle more than the rest combined,
   each with why it matters, not just what is wrong.
3. **The journey**: the timestamped persona walkthrough(s), showing where the
   page wins and where it loses.
4. **The backlog**: every finding, grouped by tier, each with the concrete fix
   and its weight. One line per finding; no essays.
5. **Unanswered objections**: the question table.
6. **Unused arguments**: including the ROI math, worked out.
7. **Dimension scores**: with a short note per dimension.

Writing rules:

- Quote the page's actual copy. "The button is unclear" is worthless; "the button
  says X and lands on Y" is actionable.
- Every finding pairs a defect with a specific fix. Never list a problem alone.
- Lead with what works before what does not, and be concrete about it. An audit
  that only criticizes gets discounted; naming the genuinely strong asset tells
  the team what to protect while they fix the rest.
- State the method's limits in a footer: viewport widths tested, date, what could
  not be verified and why. Scores are expert judgment, not analytics.

When auditing several personas or several angles, write each as its own document,
then deliver **one consolidated, deduplicated backlog** across all of them - 
noting for each finding which pass(es) it surfaced in. The consolidated backlog
is the artifact the team actually works from.

## Failure modes to avoid

- **Reviewing instead of operating.** If you did not click it, mark it
  unverified. Do not assume a button works.
- **Stopping at the first blocker.** When a defect blocks further testing, record
  it, then work around it (reload, different entry point, fresh state) to cover
  the rest of the page. If you truly cannot, say which items are unverified and
 why, that gap is itself a finding.
- **Generic persona reactions.** "A user might be confused" is not a finding.
  The persona's profession supplies the objections; use them.
- **Cosmetic-first ordering.** Color and copy polish rank below a dead button,
  always. If the tier list opens with typography, it is wrong.
- **Padding the count.** Thirty-five real findings beat sixty with filler. Merge
  duplicates; drop anything you cannot state a fix for.