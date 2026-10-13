---
name: spec-writer
description: >
  Define, write, and refine software specs and user stories optimized for AI agents (Claude Code).
  Use this skill whenever the user wants to: write a spec, define a user story, refine a feature,
  turn a vague idea into a structured task, create an AC/BR/tech-spec document, prepare a task
  for Claude Code, or says things like "help me spec this", "write the spec for", "define this
  feature", "I want Claude Code to do X", "we need to implement Y", or "I need the spec for".
  Trigger even if the user just describes a feature informally, convert it into a proper spec.
  Always use this skill before writing or reviewing specs, never skip it.
---

# spec-writer

Converts vague feature ideas or user stories into structured, AI-agent-ready specs.
Output is a `.md` file following the canonical spec template.

---

## Workflow

### Step 1: Intake

Extract from conversation context (never ask for things already stated):

- **What**: feature or behavior to implement
- **Who**: user role or system actor
- **Why**: business goal or user benefit
- **Stack signals**: language, framework, DB, test runner (if mentioned)
- **Constraints**: non-functional requirements, things NOT to do

If critical info is missing, ask ONE consolidated question. Never ask multiple rounds.

---

### Step 2: Draft the Spec

Use the canonical template below. Fill every section.
If a section has no content, write `N/A`: never leave blanks.

---

### Step 3: Refine

After drafting, do a self-review pass:

- [ ] Is the Goal one sentence, no jargon?
- [ ] Are ACs written as Given/When/Then or clear conditions?
- [ ] Are BRs non-negotiable and unambiguous?
- [ ] Does Tech Spec have exact types, not prose?
- [ ] Is Out of Scope explicit and specific?
- [ ] Are Notes for Agent imperative, not suggestions?
- [ ] Would an agent reading this have zero ambiguity?
- [ ] Does every AC have exactly one named Playwright test in the E2E table?

Fix issues found before presenting to user.

---

### Step 4: Output

Present the spec as a fenced markdown code block.
Also save to `specs/[feature-id]-[slug].md` if filesystem is available.

Ask: "Do you approve this spec, or should we adjust anything before passing it to the agent?"

---

## Canonical Spec Template

~~~markdown
# SPEC: [FEATURE-ID] - [Short name]

## Meta
- **Status:** draft
- **Priority:** high | medium | low
- **Depends on:** [FEATURE-ID or none]

---

## Goal
<!-- One sentence. What problem it solves and for whom. -->

---

## User Story
As a [role], I want to [action], so that [benefit].

---

## Acceptance Criteria
<!-- Verifiable conditions. The agent uses these to generate tests. -->
- [ ] AC-1: Given [context], when [action], then [expected result].
- [ ] AC-2: ...

---

## Business Rules
<!-- Non-negotiable rules. The agent must NEVER break these under any circumstance. -->
- BR1:
- BR2:

---

## Tech Spec

### Scope
<!-- Exact files and modules. Critical so the agent does not touch what it should not. -->
- Create: `src/...`
- Modify: `src/...`
- Do not touch: `src/...`

### Inputs
```typescript
// Exact types. No prose.
interface RequestInput {
  field: type;
}
```

### Outputs
```typescript
interface ResponseOutput {
  field: type;
}
```

### Endpoint / Entry Point (if applicable)
- Method: GET | POST | PUT | DELETE
- Path: `/api/v1/...`
- Auth: required (JWT) | public
- Rate limit: N req/min

### Edge Cases
<!-- One edge case per line. The agent generates error handling for each one. -->
- [Condition] → [expected behavior and HTTP code if applicable]
- [Condition] → [expected behavior]

### DB Changes
<!-- Exact schema or "None". -->
- None

---

## Test Plan

### E2E (Playwright): one per AC
<!-- Every AC maps to exactly one test. If an AC has no test, the spec is incomplete. -->

| Test name | AC | File |
|---|---|---|
| `AC-1: <slug describing the scenario>` | AC-1 | `tests/e2e/<feature>.spec.ts` |
| `AC-2: <slug>` | AC-2 | `tests/e2e/<feature>.spec.ts` |

Skeleton:
```ts
test('AC-1: <slug>', async ({ page }) => {
  // arrange, seed state, authenticate, navigate
  // act , perform the user action
  // assert, observable outcome matches AC
});
```

### Unit: pure functions only

| Function | File | Cases |
|---|---|---|
| `<fn>` | `lib/<module>/<file>.ts` | empty / happy / edge / error |

### Verification command

```bash
<exact command to run all E2E tests for this feature>
# e.g.: pnpm test:e2e -- tests/e2e/<feature>.spec.ts
```

Expected: all green. No skips, no `it.only`.

---

## Out of Scope
<!-- Explicit. Prevents the agent from adding unrequested features. -->
- Do not implement [X]
- Do not modify [Y]
- Do not install new dependencies without approval

---

## Notes for Agent
<!-- Direct instructions. Imperative tone. The agent follows these to the letter. -->
- Follow the existing [service/repository | MVC | etc] pattern in the project.
- Write tests BEFORE implementing.
- If you find ambiguity in a BR, stop and ask before continuing.
- Do not add extra logging or unnecessary comments.
- Commit only when all tests pass.
~~~

---

## Quality Rules

**Goal**: must be one sentence, no "and", no conjunctions.
**ACs**: minimum 2, maximum 8. More than 8 = the spec is doing too much, split it.
**BRs**: only business-dictated rules. No technical rules (those go in Notes for Agent).
**Scope**: always include "Do not touch" even if only one file.
**Edge Cases**: minimum 2. If there are no edge cases, the analysis is incomplete.
**Out of Scope**: minimum 2 items. Forces the user to think about limits.
**Notes for Agent**: always include the "stop if ambiguous" rule.
**Test Plan**: every AC must have exactly one named Playwright test in the E2E table. No table = incomplete spec.

---

## Refinement Mode

If the user brings an existing spec to refine:

1. Read the spec in full
2. Evaluate against the Quality Rules above, including:
   - Verify every AC has exactly one test in the E2E Test Plan table
   - Verify a Verification command exists
3. List issues found (numbered)
4. Present the corrected spec
5. Summarize changes made

Do not ask permission to fix, fix and show the diff.

---

## Language

- Output specs in English always.
- Technical terms (endpoint, JWT, UUID) always in English.
