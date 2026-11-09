---
name: project-index-manager
description: Queries and updates a Project Index (a living catalog in an artifact with a database) built by project-index-builder - find, register, update, replace, favorite and audit records. Use to find something in a project, to register what was just created, or to answer what is urgent today from the daily status. Works with requests in Spanish or English.
---

# Project Index Manager

Operate the Project Index created with `project-index-builder`. It is generic: the index, its domains and its brand come from the data. Answer in the user's language.

## 1. Find the index
In this order:
1. An index link in the project instructions or in its rules doc (`project_info`, `project_search`).
2. `Artifact` with `action: list` and a title containing "Index", "Map", "Índice" or "Mapa" for the project.
3. Ask the user for the link only if you cannot find it.
Read the page (`Artifact` `action: read`) the first time to learn the domains (`DOMS`) and the brand. All data reads and writes use `ArtifactData` with the index `url` and the `items` collection. If `ArtifactData` is not loaded, load it with ToolSearch.

## 2. Query
For "where is X", "what is there about Y", "what changed this week":
- `query` or `list` on `items` (use `query.limit` and `cursor`; with `out_dir` when there are many, then read the files).
- Filter by `dominio`, `tipo`, `aud`, `estado`, and search `titulo` and `nota` case-insensitively.
- Sort by `actualizado` (or `fecha`) descending.
Answer with title, domain, date and time, and the right link (`url_visual` if present, otherwise `url`). Show only current items unless history is asked for. Every reference carries its link.

## 3. Register what is new (required when you create something)
Everything you create or meaningfully change for the project is registered in the same turn, before the final answer. `set` with `doc_id` = slug `<dominio>-<titulo>-<last 8 characters of the url>` and the fields (fixed schema keys, keep them as written):
`dominio`, `titulo`, `url`, `url_visual` (optional), `tipo`, `aud`, `fecha`, `creado`, `actualizado`, `autor`, `visibilidad`, `nota`, `estado`, `clave`, and for favorites `inicio` (order 1, 2, 3...; the first 6 are "Start here"), `inicio_label` and `inicio_desc` (optional for the card).
`tipo` uses the builder's canonical values (`Document`, `Design`, `Animation`, `Design system`, `Spreadsheet`, `Presentation`, `Dashboard`, `Whiteboard`, `Tasks`, `Meeting`, `Code`, `Tool`, `Folder`; existing Spanish values are also valid) and comes from the artifact's real type (`Artifact` `list` with `type`), never from the title.
Field rules:
- `actualizado` in ISO with the real time (`YYYY-MM-DDTHH:MM:SSZ`, UTC) when known; the page shows it in local time. Without a time, the day only; if unknown, empty.
- Never invent metadata: what you cannot confirm stays empty and shows as "To verify".
- `clave: true` only if it becomes the domain's entry document; the previous one goes to `false`.
- A doc that lives only in the Claude Project has an empty `url`.
- `aud` is `cliente` (client), `interna` (internal), `conf` (internal confidential) or `colaborador` (collaborator); `estado` is `vigente` (current) or `anterior` (earlier version).
If the domain does not exist in the index, use the closest one and say so; do not create domains unless the user asks.

## 4. Update, replace, link
- **Updated**: find the record by `url`, read its `version`, and `update` `actualizado` (with the real time) and `nota` if it changed, with `if_version`.
- **Replaced**: the old one goes to `estado: "anterior"` and `clave: false`; the new one enters as `vigente`.
- **Visual version**: if a repo doc has its own design or artifact, put its direct link in `url_visual` (with the exact `#page-...` or `#play-...`). The index opens it instead of the source file and keeps the source as secondary.
- **Favorites**: to add one, `update` the record with `inicio` = current highest number + 1 (or the order requested). To remove it, `update` with `inicio`, `inicio_label` and `inicio_desc` set to `{"__delete__": true}`. To reorder, renumber `inicio` 1 to n. The first 6 form "Start here"; if a favorite is replaced, move its `inicio` to the new record.
- Several writes: `batch` (up to 50), each entry with its `if_version`. If a version changed, re-read and redo that write; never overwrite blindly.
- **Never delete records.** Obsolete items are marked `anterior`.

## 5. Audit
When asked to review the index, or when you notice disorder, report without fixing unless given permission:
- Duplicates (same `url` or same title), records without a `url` that should have one, broken or non-`https` URLs.
- Empty fields that can be verified (visibility, author, dates) and how to verify them.
- Domains with more than one `clave`, or none.
- Current items not updated for a long time, and favorites pointing to `anterior` versions.
- Artifacts or docs of the project that exist and are not registered.
Propose the changes as a short list and apply them only after confirmation, using `batch` with `if_version`.

## 6. Change the page
Content lives in the database; do not republish to add records. For page changes (design, domains, brand), edit the source file and republish with the `Artifact` tool to the same URL; load `artifact-design` first and keep the `db` and `mcp` capabilities (omit `capabilities` when republishing so they are kept; if you declare them, declare the full set).

## 7. Daily status
If the index has the daily status view, its data lives in the `status` collection, one document per day (`YYYY-MM-DD`) written by `client-status-update` in daily mode.
- For "what is urgent", "what is next", "who has what" or "how are we doing today", read the latest document and answer with its blocks; give the record's date and say if it is not today's.
- Do not edit past days. To correct today's, rerun the daily mode or `update` with `if_version` and note the correction in `notes`.
- Do not register daily statuses as `items`: they already live in the index.

## Close
Short answer: what you looked up or registered, how many records, and what was left "To verify". If you could not register something (no `ArtifactData` or no access), say exactly what is missing, with all its fields, so it can be registered later.
