<p align="center"><img src="assets/logo.png" alt="DashOne" width="120"></p>

# DashOne Community Plugins

Skills from DashOne that work for any team. Install the whole plugin, not skill by skill.

## Install

```
/plugin marketplace add DashOne-AI/community-plugins
/plugin install dashone-community@dashone-community-plugins
```

## What is inside

Plugin `dashone-community`:

| Skill | What it does |
|---|---|
| `diagnose` | Diagnoses a production bug, alert or incident: root cause with evidence, impact, severity, priority and a fix plan. |
| `handoff` | Appends a handoff entry to `Handoff.md` so work can resume in a fresh session after `/clear`. |
| `loop-instructions-builder` | Turns an approved plan into an execution-authorization prompt, checked against Claude Code's real hard limits. |
| `loop-from-plan` | Takes an approved plan, creates one GitHub issue per phase, and launches its autonomous execution with `/loop`. Pairs with `loop-instructions-builder`. |
| `spec-writer` | Turns an idea into a spec with user stories and acceptance criteria, ready for an AI agent. |
| `doc-reader` | Finds and reads any document aloud-friendly, full or summarized. |
| `landing-page-conversion-analysis` | Audits a landing page by walking it as real buyer personas and delivers prioritized findings. |

## Optional companion: graphify

No skill here requires it. `graphify` turns a folder or repo into a navigable knowledge graph and pairs well with `diagnose` and `spec-writer` when you need architecture context.
It is third-party (Apache-2.0 and MIT, [Graphify-Labs/graphify](https://github.com/Graphify-Labs/graphify)) and is not bundled, so you get its updates from the source:

```
uv tool install --upgrade graphifyy
graphify install
```

Check it with `graphify --help`. Skip this section if you do not need codebase graphs.

## Notes

- All skills are written in English and work with requests in Spanish or English.
- These skills carry no client data and no credentials.
- License: MIT. See `LICENSE`.
