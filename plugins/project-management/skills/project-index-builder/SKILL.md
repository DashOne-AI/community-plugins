---
name: project-index-builder
description: Builds a Project Index, a navigable artifact that catalogs everything a project holds (docs, artifacts, designs, repos, Drive, meetings) by domain, with an icon and color per artifact kind, a shared database and an optional Linear daily status. White label. Use when someone asks to create, build or set up a project index, project map or catalog of a project's files. Works with requests in Spanish or English.
---

# Project Index Builder

Build a published artifact (page plus shared database) that works as a living index of a project. The content lives in the database, not in the page: any session can add or change records without republishing. It is generic: assume no client, brand or industry. Everything that changes between projects goes in a brand configuration (`BRAND`) and in the data.

Answer in the user's language. To operate an index that already exists, use the `project-index-manager` skill.

Goal: do as much as possible in one run. Read every reachable source without asking, register everything that clearly belongs, and leave the user only the decisions that are genuinely theirs (doubtful items, pasting instructions, approving scheduled tasks).

**Tool names.** This skill says `Artifact` and `ArtifactData`. When the session instead has `mcp__claude-artifacts__*` tools, map them: list artifacts → `list_artifacts` (`query`, `type`, `cursor`); artifact types → `list_types`; read state → `get_artifact`; publish the page → `publish_files` (or `publish_version`); declare capabilities → `declare_capabilities`; write records → `db_write` (`ops` with `set`/`merge`/`update`, path `items/<doc_id>`); read records → `db_list` / `db_get` (`asLevel: "view"` to test a reader). Order for a new page: publish first, then declare capabilities (the declaration republishes the page), then write records.

Reference implementation: `assets/template.html`, next to this file. Read it, copy its code and change only `BRAND`, `LAYERS` and `DOMS`. It already implements everything described below: layout, favorites, details panel, search, per-kind icons and colors (5a), Daily status and Board. Drop the `status` block from `BRAND` when there is no Linear project. If a change of structure or style is requested on an index, apply it to this skill and the template too, so the next index is born the same way.

## 1. Default scope
Index whatever is where the skill runs, without asking permission to read:
1. The Claude Project attached to the session (`project_info`, `project_search`, its docs and synced sources).
2. Synced or connected sources: repo (README, folder tree, docs, minutes), Drive (folders and files), Linear or another tracker (as one entry, not its tickets), the team's connected folders.
3. Published artifacts in the account related to the project (`Artifact` with `action: list`; filter by title and by links that appear in the project docs).
4. Links mentioned in the project docs (earlier indexes, handoffs, instructions).
5. Every conversation of the project, always (not only when the project is empty): page through `recent_chats` (it is scoped to the project) and register each relevant chat as `tipo: Meeting` in the sources domain, with its last update time. Also run `conversation_search` for `claude.ai/artifact` and for the project's names inside those chats to harvest artifact links.
6. Artifacts moved into the project are not returned by `project_info`. Find them by: links harvested from the project's chats and docs; `list_artifacts` queried by the project name and by every alias found in memory (renamed projects, related products) and paged to the end; links inside artifacts already found (a whiteboard or index that links to others). An artifact that `get_artifact` reports as `outside_organization` cannot be read: register it anyway with an honest descriptive title, empty `tipo` and the note "title and type to verify".
7. The tracker: search Linear projects and documents by the project name and its aliases. If a Linear project of its own exists, register it and turn on Daily status and Board (5b) without asking. Register related projects' Linear entries only when the project's material links to them.
8. The team's context repo, if memory or the project docs name one (for example a private context repo reached through a GitHub connector such as Composio when the git proxy cannot attach it): look for the project's folder, register its files with their frontmatter, and if there is no folder say so in the close.
If the attached project is empty and nothing else turns up, build the inventory from the account's artifacts and the project's conversations, and say so in the close.
If the user names another scope (another folder, project or repo), use it instead of the default.

Ask only what you cannot infer, in one batch: the project name for the title, a logo or colors if they want their own brand, and whether there are audiences to separate (for example client vs internal). Decide everything else and state it as an assumption. Use the project name exactly as the user says it; if they rename it later, change `BRAND.name`, the `<title>` and the texts that repeat it, but leave the titles of indexed artifacts alone (they are their real names).

## 2. Inventory
Walk the sources and build a flat list. For each element gather only what you can verify:
- title, url (empty if it lives only inside the Claude Project), type, author or owner, creation and last-modified dates (with time when the tool gives it), visibility (public, organization, private) only if the tool says so.
- Read the frontmatter of repo docs when it exists (title, description, audience, owner, last review, visual version).
- Doubtful items: anything that may or may not belong to the project (same family of products, shared people, ambiguous title) goes into a short list for the close, with one line of why. Do not drop it silently and do not register it without saying so.
- Deduplicate: same url, copies with the same title (mark the copy as `anterior` and say so), superseded versions (`estado: anterior`).

Never invent metadata. What you cannot confirm stays empty and the page shows it as "To verify". A classification opinion is yours and you declare it; data from a tool is a fact.

**How to decide the type of a published artifact (required):**
1. Call `Artifact` once with `action: list`, `scope: types` to see the account's artifact types.
2. For each type that exists (Docs, Design, Motion, Slides, Dashboard, Design System, Sheets, Whiteboard, Tasks), call `Artifact` with `action: list` and `type: <type name>` (limit 50) and match the URLs against your inventory. An artifact that appears in a type's listing is of that type: a fact, not a judgment.
3. If a type's list is cut off (it says there are more than 50), read the artifact with `action: read` and a short `prompt`, or keep it as a declared judgment.
4. An artifact that appears in no type is a hand-built page: `Design` if it is a page or site, `Tool` if it is a calculator or utility. Declare that classification as a judgment.
5. For anything that is not an artifact (repo files, Drive, meetings), infer the type from the extension or the source.
Never classify by title (a "Product Sheet" can be a Design canvas, not a spreadsheet).

## 3. Domains (the grouping that matters)
Group by domain, meaning which part of the project an element serves, not by source folder or tool. Derive the domains from the real content (4 to 9 when there is enough; for a small project, 3 well-defined domains beat 4 invented ones, and you say so). Starting template, adapt and rename it:
- Direction: `governance` (indexes, working rules, tracker, charter, folders).
- Agreement: `commercial` (proposal, contract, agreed scope), `product` (PRD, requirements, glossary).
- Operation: `process` (how the business works today), `data` or `pricing` (sources of truth for numbers), `agents` or `automation`.
- Build: `architecture` (technical decisions, repos), `experience` (design, brand, prototypes, animations).
- Sources: `meetings` (minutes, transcripts, recordings).

Each domain has a short name, a question it answers ("What was agreed?"), its layer and an icon (a key of `BRAND.typeIcons`). Inside a domain, group by artifact kind in this order: Documents, Designs, Animations, Design systems, Spreadsheets, Presentations, Dashboards, Whiteboards, Tasks, Meetings, Code, Tools and folders. Show only groups that have elements.
Doubtful cases: pick one and list it in the final answer as a judgment, so the user can correct it.

## 4. Data model (collection `items`)
One document per element. `doc_id` = slug `<dominio>-<titulo>-<last 8 characters of the url>`, lowercase, letters, numbers and hyphens only. Generate the ids and the whole `ops` payload with a short script (strip accents, slug, cut the title part to 50 characters, take the url's last 8 characters) instead of writing them by hand.

Field names are fixed keys of the schema shared by the page and `project-index-manager`; keep them exactly as written, even though they are in Spanish:
`dominio` (domain id), `titulo` (title), `url`, `url_visual` (optional, direct visual version), `tipo` (type), `aud` (audience: `cliente` client | `interna` internal | `conf` internal confidential | `colaborador` collaborator), `fecha` (document date, ISO), `creado` (created), `actualizado` (updated; ISO with time when known, `YYYY-MM-DDTHH:MM:SSZ`; otherwise the day only; otherwise empty), `autor` (author), `visibilidad` (visibility: `publico` | `organizacion` | `privado` | `personas` shared with specific people, as the tool reports it), `nota` (one-line note, no sensitive figures), `estado` (`vigente` current | `anterior` earlier version), `clave` (bool: the domain's entry document, one per domain).

`tipo` uses one of these canonical values (the page also recognizes Spanish names and text without accents): `Document`, `Design`, `Animation`, `Design system`, `Spreadsheet`, `Presentation`, `Dashboard`, `Whiteboard`, `Tasks`, `Meeting`, `Code`, `Tool` or `Folder`. Map from the account's artifact types: Docs → Document; Design → Design; Motion → Animation; Design System → Design system; Sheets → Spreadsheet; Slides → Presentation; Dashboard → Dashboard; Whiteboard → Whiteboard; Tasks → Tasks.

Favorites: `inicio` (optional) marks the record as a favorite and is its order (1, 2, 3...). The first 6 form "Start here" on the home view; all of them appear in the Favorites view. Optional for the card: `inicio_label` (short label) and `inicio_desc` (one line; falls back to `nota`). To remove a favorite, delete `inicio`, `inicio_label` and `inicio_desc` with `{"__delete__": true}`; to reorder, renumber 1 to n.
Propose the first favorites as the documents that tell the project in order: what is being built, what was agreed, how it works today, how it is built. If the project lacks some, use what exists and do not pad.

## 5. The page (artifact)
Before writing, load the `artifact-design` and `artifact-capabilities` skills. Publish with the `Artifact` tool, capability `db` with root rule `{path: "", read: "view", write: "admin"}`. The page subscribes with `db.collection("items").onSnapshot(...)` and renders everything from the data (never embed the records in the HTML). With no data it shows a useful empty state. Documents from the database arrive frozen: copy them before adding fields (`Object.assign({}, d.data())`), and mark loading as finished also when `db` is unavailable or the subscription fails, so the view never stays on "Loading".

White label: define a `BRAND` object at the top of the script with `name`, `title`, `sub`, `project` (Claude Project where docs without a link live), `logo` (optional data URI; if empty, a text wordmark with the name), `homeTitle`, `footer`, `signin`, interface texts in `ui`, `kinds`, `typeIcons`, `storePrefix` (prefix for `localStorage` keys), color tokens (light and dark) and fonts (Google Fonts allowed). Leave no client, product, person, workspace, repo or project URL outside `BRAND` and the data. If the user already has a design system (memory, project), use it for the base tokens.
Every visible text lives in `BRAND.ui`, in the project's language: navigation, home, panel, filters, plurals with `{n}` (use `tr`/`trn` helpers), favorite and error messages, and the words the favorites field recognizes (`nlRemoveWords`, `nlStopWords`, one regex alternative per word). Set `<html lang>` to the project's language.

Behavior:
- Two-pane layout: side navigation with, at the top, Home, Favorites (with a counter) and, if there is status, Daily status (with the date of the latest record) and Board; below, layers and domains with a count of current items. The sidebar takes the full height (`height: 100dvh` minus safe areas, `position: sticky`). Mobile: collapsible navigation.
- Home: title, a sentence with the total of current items and domains, "Start here" with the first 6 favorites as cards (3-column grid that drops to 1 on mobile), the Daily status strip if it exists, and the "New or updated this week" list.
- Favorites: a dedicated view with all favorites in order, arrows to move up and down (they renumber `inicio` 1 to n) and a star to remove. Every element row and card has a star (`aria-pressed`, `aria-label` "Add to favorites" or "Remove from favorites") that writes `inicio = max + 1` or deletes the fields. The view also has a text field to ask in natural language ("add the PRD to favorites", "remove Monday's minutes"): first search the words in title, note and type; if nothing matches, use the `sample` capability (`json`) over the list of titles to pick ids; with several matches show the list to star. Writes need admin level; if one fails, say so.
- Domain view: header with the domain's icon and question, card for the `clave` document, audience filter (only when there is more than one audience), groups by artifact kind, order by `actualizado` (or `fecha`) descending with the time in local time when it exists, earlier versions collapsed.
- Row: kind icon on the left (see 5a); the title is the direct link (opens `url_visual` if present, otherwise `url`, in a new tab) and the whole row is clickable to that link. Under the title go the note, the "new" indicator, the audience when it is client or confidential, and the date with time. Next to the title go the favorite star and an info button (ⓘ, `aria-label` "Details: <title>", touch target of at least 40 px), the only control that opens the side panel with Author, Created, Updated, Visibility, Audience, Type, Where it lives, Status and Link; if there is `url_visual`, show the source separately. Empty values read "To verify". Without a link the title is not a link (dimmed, no navigation) and the ⓘ stays available. No ↗ at the end of the row.
- Cards (Start here, favorites, key document and any other card that shows an element): same rule. The main action goes straight to the link, the ⓘ opens the panel, and the card also carries the kind icon with its color. Search and home lists use the same row component.
- Pattern to avoid nesting interactive controls: the title link stretches over the whole row and the buttons sit above it.
  HTML: `<li class="row"><span class="ftype">svg</span><div class="body"><div class="titleline"><a class="row-title" href="..." target="_blank" rel="noopener noreferrer">Title</a><button type="button" class="info" aria-label="Details: Title">svg</button></div>note and meta</div></li>`
  CSS: `.row{position:relative}` `.row-title::after{content:"";position:absolute;inset:0}` `.info,.fctl{position:relative;z-index:1}`. Visible focus on every control; focus returns to the ⓘ when the panel closes.
- Render: save the `data-fid` of the focused element before each render and restore focus after, so clicks and keys do not lose it.
- Global search ("/" shortcut), accessible, discreet inline SVG icons, no external libraries except fonts.
Unknown domains fall into the first domain of the first layer (`LAYERS[0].doms[0]`), never into a fixed id. All links `https`. Escape every value that reaches the HTML.

### 5a. Icons and colors per artifact kind (required)
Each element shows an inline SVG icon (stroke 1.5, 20 px, `aria-hidden`) inside a 36 px box with 8 px corners, tinted with its kind's color (`background: color-mix(in srgb, currentColor 13%, transparent)`, `color: var(--k-<kind>)`). The kind's name goes as hidden text (`.sr`) and as `title`, so meaning never depends on color or icon alone.

`BRAND.kinds` is the ordered list of kinds `{id, group}`; `BRAND.typeIcons` uses the same ids as keys, plus `info`. Kinds, group, icon and color (light / dark), all as `--k-<id>` tokens defined in `:root` and redefined in both dark blocks:

| id | Group | Icon (content of `<svg viewBox="0 0 24 24">`) | Light | Dark |
|---|---|---|---|---|
| doc | Documents | `<path d="M6 3h8l4 4v14H6z"/><path d="M14 3v4h4M9 12h6M9 16h6"/>` | #2F6FB5 | #74A7EC |
| design | Designs | `<rect x="6" y="6" width="12" height="12" rx="1"/><path d="M2 9h4M2 15h4M18 9h4M18 15h4M9 2v4M15 2v4M9 18v4M15 18v4"/>` | #D9480F | #FF8A5C |
| animation | Animations | `<rect x="3" y="5" width="18" height="14" rx="2"/><path d="M10 9.5v5l4.5-2.5z"/>` | #C2255C | #F06595 |
| system | Design systems | `<rect x="4" y="4" width="7" height="7" rx="1"/><circle cx="16.5" cy="7.5" r="3.5"/><path d="M7.5 13l3.5 7h-7zM14 14h6v6h-6z"/>` | #7048E8 | #A78BFA |
| sheet | Spreadsheets | `<rect x="4" y="4" width="16" height="16" rx="1"/><path d="M4 10h16M4 15h16M10 4v16"/>` | #2B8A3E | #5CC27A |
| slides | Presentations | `<rect x="3" y="5" width="18" height="12" rx="1"/><path d="M8 21h8M12 17v4"/>` | #C27803 | #F2B13D |
| dashboard | Dashboards | `<path d="M4 20h16M7 20v-6M12 20V6M17 20v-9"/>` | #0C8599 | #4CC3D9 |
| whiteboard | Whiteboards | `<rect x="3" y="4" width="18" height="12" rx="1"/><path d="M8 21l2-5M16 21l-2-5M7 9h4M7 12h7"/>` | #9C6B30 | #D0A35E |
| tasks | Tasks | `<path d="M4 7l2 2 3-3M4 16l2 2 3-3M12 8h8M12 17h8"/>` | #5C940D | #A3D95A |
| meeting | Meetings | `<rect x="4" y="5" width="16" height="15" rx="2"/><path d="M4 10h16M9 3v4M15 3v4"/>` | #12806A | #4FC3AA |
| code | Code | `<path d="M9 8l-4 4 4 4M15 8l4 4-4 4M13 6l-2 12"/>` | #5C6370 | #A5ACBA |
| folder | Tools and folders | `<path d="M3 7a1 1 0 011-1h5l2 2h8a1 1 0 011 1v9a1 1 0 01-1 1H4a1 1 0 01-1-1z"/>` | #7D8597 | #A5ACBA |

The kind color is independent of the brand accent: the accent is reserved for focus, active navigation and primary actions. If the brand already uses one of these hues as its accent, shift the kind's hue instead of repeating it.

Resolving `tipo` to a kind in the page: normalize the text (lowercase, no accents) and walk this table in order; the first match wins, and with no match the kind is `doc`:
`system` /design system|sistema de diseno|tokens/ · `animation` /animation|animacion|motion|video|film/ · `dashboard` /dashboard|tablero|panel/ · `whiteboard` /whiteboard|pizarra/ · `tasks` /task|tarea|to-do|todo/ · `meeting` /meeting|reunion|minute|minuta|transcri/ · `code` /code|codigo|repo/ · `slides` /presentation|presentacion|slides|deck|slide/ · `sheet` /spreadsheet|sheet|hoja|table|tabla|data|dato|csv/ · `folder` /tool|herramienta|folder|carpeta|tracker/ · `design` /design|diseno|page|pagina|poster|mockup|landing|canvas/.
Order matters: "Design system" must resolve before "Design".

### 5b. Daily status and Board (optional, when the project uses Linear)
Add them when the user wants a daily status or the project has a Linear tracker. This part's interface is in English unless the user asks for another language.

Data and configuration:
- Collection `status`, one document per day (`doc_id` = `YYYY-MM-DD`) written by `client-status-update` in daily mode. The page subscribes with `db.collection("status").onSnapshot(...)` and sorts by date descending. Copy `data()` before adding fields, in `items` and in `status`.
- Accept the shapes the daily mode writes: `assign` may hold bare ids (`"ABC-30"`) as well as `{id, title}`; normalize them on read. The `changes` points (`closed`, `opened`, `moved`, `updated`) may come as `{text, owner, refs}` or as `{id, url, title}`; show both.
- In Linear, `statusType: "duplicate"` counts as closed like `canceled` (normalize it when receiving `list_issues`), so it does not inflate open, overdue or the board.
- `BRAND.status`: `linearServer` (connector name), `linearProject` (slug), `linearWorkspace` (for `https://linear.app/<workspace>/issue/<id>`; every issue link comes from one `issueUrl` function), `projectName`, `projectUrl` (Linear project), `indexUrl` (this index), `codeRepo` (`owner/repo` opened by the Code button; empty hides the button), `scheduleName` and `scheduleTime` (the scheduled task that regenerates the status), and `people` (`[{name, match: [Linear user names, emails or ids], assignee: <Linear user id>}]`; clients go without `assignee`).
- Regeneration: offer to create a daily scheduled task at 21:59 in the user's local time that runs `client-status-update` in daily mode and writes the next day's record (`doc_id` = tomorrow), with the same shape as the latest, without Done or Canceled issues and without changing anything in Linear. The page explains this only in the "out of date" tooltip.

Live layer (Linear live):
- Capability `mcp` with the connector's `list_issues` and `save_issue`. `watchTool` over the project's `list_issues` (`limit` 250; `fields` title, status, statusType, dueDate, assignee, labels, description; `refetchInterval` 5 min), started when Daily status or Board opens, or on load if `permissions.state("mcp:<linearServer>")` is already `granted`; never ask for permission when Home loads. On `visibilitychange` back to the tab, refresh if the data is older than 2 min. The refresh button calls `callTool` with `{cache: {refresh: true}}`. Freshness comes from `cache.storedAt`.
- Cross with the snapshot: each point of the day's record is matched to Linear through its `refs`. If all its issues are closed, it shows struck through with "Done in Linear" (or "Canceled in Linear") until the next run; if they are still open, lateness is recomputed to today with the live date. Open, overdue and unassigned counters come from the live data.
- Writes: "Mark <ID> done" (`save_issue {id, state: "Done"}`, confirm with the response's `statusType`, Undo returns to the previous state), assign (`save_issue {id, assignee}`, confirm with the returned `assignee`, Undo with `assignee: null`) and move state (`save_issue {id, state: <name>}`, confirm with `status`). The page changes only after Linear confirms; then `invalidate` `list_issues`. Each error code gets its own text (reconnect, add the connector, allow access with `permissions.request`, policy, connector error); on `server_unavailable` or `upstream_error` say Linear did not confirm and to check the issue before retrying. Keep the last data on transient errors. Before publishing, observe a real `list_issues` response and a no-op `save_issue`; if you cannot, say so.
- Always declare the full set of capabilities: `db` with its rules, `mcp` with `list_issues` and `save_issue`, and `sample`.

Daily status view, in this order and nothing else visible:
1. One-line header: icon and "Daily status", no description below. If the latest record is older than today, a discreet red dot next to the title (28 px button, `aria-label` "Snapshot out of date") with a tooltip: snapshot date, age in days, that issues closed in Linear already show as done, and what to do: wait for tonight's run or ask Claude to run the `scheduleName` task. On the right, an icon-only refresh button (`aria-label` "Refresh from Linear") with a tooltip showing the time of the last live load. Linear errors go on one line under the header, with "Allow access" when it applies.
2. Day tabs (up to 15; only when there is more than one day).
3. Summary: health as a word (not color), date, large headline, health reason and short counters (open, overdue, unassigned, changed).
4. `<details>` accordions, all closed by default, with a short summary on the title line, stacked 12 px apart; keep which ones are open in page state so they survive every render:
   - Urgent today (only if there are urgent items; "N items · K done"): numbered cards with title, why, person, date or days late, issues as links and the record's actions (`actions`). `chat` is the primary button and opens `https://claude.ai/new?q=` with the encoded prompt, plus an icon-only copy button (if the clipboard fails, show the text to copy by hand). `draft` asks for the draft with `sample` (`modelTier: "quick"`, `cache: false`, only on click), shows it inline with Copy and Close, and hides when there is no permission. `link` opens the issue. Inline assignment if the point has `assign`.
   - This week (only on the latest day; "N issues · K done · M overdue · <week range>"): an "Open board" button and a table per assignee with a progress bar (done of not canceled), count per state, overdue, next and last due date, and a total row. Without the live layer, show "Next 5 business days" from the snapshot with a note to refresh.
   - Milestones (only if there are any; "N milestones · K at risk or late"): progress bar, Linear date against committed date, and health as a word.
   - Full status: live layer per person (open, overdue, due in 5 business days, with `owner_overrides`), Waiting on others, Next 5 business days per person, Decisions, Missing or wrong, Since yesterday, and a footer with sources, notes and generation time.
   Empty block: "Nothing here today."

Board view (full screen; hides header and navigation, `#board` in the URL, Escape returns to Daily status):
- Header: "← Daily status", icon and "Board", icon refresh. Controls: group by Assignee or Week (segmented), "Show/Hide Backlog" with count, a "N cards · overdue · blocked" summary on the right, and a checkbox per state with its count. Board preferences and hidden states in `localStorage` (with try/catch); Done and Canceled hidden by default.
- Set: everything open plus what closed with a date inside the week. Lanes per person (order of `people`, Unassigned last) or per due week (Before this week, This week, Next week, Week of <date>, No due date). Columns per state in flow order (Triage, Backlog, Todo, In Progress, In Review, Done, Canceled).
- Flat card: ID as a discreet link, a Blocked mark only when Linear says so (label or state containing "block"), an assignee badge that is also a `<select>` to reassign or unassign (color per person, 4 hues with a dark version), title in normal weight on 2 lines, date or "<date> · Nd late" in red, and actions that appear on hover or focus (always visible on touch). Tooltip with the first useful line of the description. Dragging a card to another column, or Alt + ←/→ with focus, changes the state in Linear; message "Moved from X · Undo".
- Actions per issue (on the Board and in any issue list, on the same line as the information, on the right, wrapping on narrow screens): "Code" (primary, hidden on closed issues or without `codeRepo`) opens `https://claude.ai/code?prompt=<...>&repositories=<codeRepo>` with ID, title, link, assignee, date, state and the instruction to read the issue and its comments, propose the plan, implement it and update the issue; "Ask" (secondary) opens `https://claude.ai/new?q=<...>` with the same context plus `projectUrl` and `indexUrl`. Do not use claude.ai project links for "Ask": they open the project but ignore the prompt.
- Color per state (dot and text, never color alone), tokens in light and dark: Triage orange, Backlog light gray, Todo bluish gray, In Progress blue (the accent), In Review purple, Done green, Canceled pale gray. Filter checkboxes are simple: a checkbox with the state's `accent-color`, the name in its color and the count, no pill border or dot.
- Custom tooltips (not `title`): they appear on hover and on focus, ink background, 12.5 px text, max width 300 px, right-aligned when the trigger is at the right edge.

Home: the status strip shows health, date, headline, numbered urgent items (struck through if already closed in Linear), "Open Daily status" and refresh.

## 6. Loading the data
After publishing, load the records with `ArtifactData` `batch` (up to 50 per call, `set`, no `if_version` because they are new). Check with `list` that the count matches the inventory, repeat the `list` with `as_level: view` to confirm a view-only reader sees them, and open the page.
For later small changes (renaming, icons, texts), republish the same file with `edits` (old_str/new_str pairs) instead of rewriting it, and update records with `update` and `if_version`.

## 7. Keep it alive
The index is only useful if it is always updated. Do all of this in the same run:
1. **Rules doc, always.** Write `claude/project-index.md` into the attached Claude Project with `project_write` (replace it if it exists). It holds the index URL and slug, the registration rule below, the domains with their ids, and the field format from section 4 (favorites and canonical `tipo` values included). Rule:
   "Required registration: every artifact, doc, design, file, minutes or repo you create or change for this project is registered in the same turn in the Project Index (<index URL>) with ArtifactData (db_write on slug <slug>), collection items, using the project-index-manager skill. If you do not register it, the task is not done."
2. **Project instructions.** The Projects tool can read the instructions but not write them. Give the user one fixed line to paste once, which points to the doc so it never needs editing again: "Before creating or changing anything in this project, read `claude/project-index.md` and follow its registration rule." If `project_info` shows the line is already there, say nothing.
   If a browser tool that acts on the user's own claude.ai session is available (Claude in Chrome or the built-in browser), offer to paste it for them: open the project's settings on claude.ai, add the line at the end of the instructions without removing anything, and save only after the user approves the exact text. Never do it without that approval.
3. **Scheduled tasks.** Offer both in one question, and create the ones the user accepts with the session's scheduling tools (never local cron):
   - Daily audit: compare the project's chats and the account's recent artifacts (same discovery as section 1, items 5 and 6) against `items`, register what clearly belongs, and send the user a short list of doubtful items.
   - Backup: export `items` (and `status` if it exists) as `index.json` into the team's context repo, in the project's folder, committing only when something changed. If the repo has no folder for the project, ask where it goes before creating the task.

## 8. Close
Answer briefly: which index you created, how many records and domains, which classification judgments are worth reviewing (including `tipo` values you could not confirm against an artifact type), the doubtful items to decide on, what was left "To verify" (for example visibility), the line to paste into the instructions (unless already there), and the scheduled tasks created or pending. Do not paste the URL if the interface already shows the card.

## Rules
- Never delete records or invent metadata.
- No secrets, tokens or sensitive personal data in notes.
- Internal material is `aud: interna` or `conf`; do not promote it to `cliente` without evidence.
- `tipo` comes from the artifact's real type, not its title.
- If the project already has an index, do not create another: use `project-index-manager`.
- Before republishing an index, read it if another session could have changed it, and build on the live version.
