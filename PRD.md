# PRD — `atl`: Atlassian MCP Server + CLI

## 1. Summary

Single Go binary (`atl`) that talks to self-hosted Atlassian **Server / Data Center** REST APIs (Jira, Confluence) and exposes the results in two ways:

1. **MCP server** (stdio) — consumed by GitHub Copilot app, VS Code Copilot, Copilot CLI, or any MCP-capable client.
2. **CLI** — the same operations as human-invokable subcommands.

The core value is not raw API access but **structured, LLM-friendly output**: issues and pages are normalized into compact, predictable Markdown/JSON instead of the raw multi-kilobyte Atlassian payloads.

The tool is **strictly read-only toward Atlassian** — it never modifies anything in Jira or Confluence. Safe to hand to anyone. (It does maintain local, LLM-adjustable instance mappings and notes — §6b — but those never leave the user's machine.)

## 2. Problem

People working with self-hosted Jira/Confluence want their AI assistant (Copilot) to answer questions and drive routine workflows without opening the Atlassian UIs:

- **Developers**: "what's in PROJ-1234?", "find open bugs in this sprint", "pull up the design page for this feature".
- **Business analysts / product owners**: "what's in the current sprint and what's its status breakdown?", "what didn't finish last sprint?", "show the backlog for board X" — i.e. read and analyze sprint data conversationally to plan sprint organization (changes themselves are made in the Jira UI).

Raw REST responses are huge, noisy, and undocumented for the model; there is no good MCP option for **Server/DC** instances (most existing tools target Atlassian Cloud, whose APIs and auth differ).

## 3. Users & distribution constraints

- End users are **developers, business analysts, and product owners** using **Copilot app / VS Code Copilot / Copilot CLI**. BAs/POs are not assumed to be comfortable with terminals: install must be download-binary + paste-config, and the primary interaction surface for them is the Copilot chat (MCP), not the CLI.
- They **cannot be assumed to have Python, Node, or any runtime installed**.
- Therefore: ship a **single static binary per platform** (darwin/arm64, darwin/amd64, windows/amd64, linux/amd64). No installer dependencies. `go install` optionally supported for Go users.
- MCP registration is one JSON snippet pointing at the binary; `atl mcp-config` prints it ready to paste.

## 4. Goals

- G1: Fetch and structure a Jira issue by key in < 2 s, output small enough to fit comfortably in an LLM context.
- G2: Search Jira via JQL and Confluence via CQL/text — with pagination handled and results summarized.
- G2b: Sprint & delivery insight for BAs/POs — boards/sprints/backlog, epic and release rollups, velocity trend, sprint scope changes — with at-a-glance aggregates (status breakdown, story points, carry-over) that hold correct totals regardless of row caps.
- G2c: Requirement traceability for BAs — from a Confluence spec page to the Jira issues implementing it (and back via issue remote links), with no silently dropped references.
- G3: Auth via **PAT (Bearer)** or **username + password (Basic)** against Server/DC. No OAuth, no Cloud API tokens.
- G4: Same feature set available identically as MCP tools and CLI subcommands (one internal service layer, two frontends).
- G5: Zero-config beyond a small config file / env vars; Jira and Confluence may live on the same or different hosts.

## 5. Non-goals (v1)

- Atlassian **Cloud** support (different APIs, different auth) — explicitly out of scope; architecture must not preclude it later.
- **Any write operations — the tool is strictly read-only.** No comments, no transitions, no sprint changes, no page edits. Only GET requests leave the client (plus POST solely where Atlassian requires it for search, e.g. none in v1 — JQL/CQL go via GET).
- **Bitbucket** — out of scope entirely (v1); architecture must not preclude adding it later.
- Attachments/binary content download.
- Caching layer, local index, or webhooks.
- A GUI.

## 6. Authentication & configuration

- Auth modes per instance: `pat` (`Authorization: Bearer <token>`) or `basic` (`Authorization: Basic user:pass`).
- Config resolution order: flags → env vars (`ATL_JIRA_URL`, `ATL_JIRA_PAT`, `ATL_JIRA_USER`/`ATL_JIRA_PASS`, same pattern for `CONFLUENCE`) → config file (`~/.config/atl/config.toml` on macOS/Linux, `%APPDATA%\atl\config.toml` on Windows).
- Config is normally created by the `atl init` wizard (§7 cross-cutting); hand-editing is the power-user path, not the expected one.
- Config file defines the two product base URLs and credentials; products may share a host or differ.
- Secrets never logged, never echoed in errors, never included in MCP tool output.
- TLS: verify by default; `insecure_skip_verify` opt-in per instance for internal CAs (with a startup warning), plus `ca_bundle` path option.

Example config:

```toml
[jira]
url = "https://jira.internal.example.com"
auth = "pat"
token = "..."          # or user = "..." / password = "..."

[confluence]
url = "https://confluence.internal.example.com"
auth = "basic"
user = "jdoe"
password = "..."
```

### 6b. Adaptive instance configuration (LLM-adjustable)

Every Jira/Confluence instance is a snowflake: story points live in a different `customfield_*`, epics link differently, teams have board aliases, workflow states carry local meaning ("Blocked-Ext means waiting on vendor"). The tool ships a **second, LLM-writable config layer** so the agent can learn these once and reuse them across sessions:

- **Two files, strict separation.** `config.toml` (credentials, URLs, TLS) is **never writable at runtime** — not by any MCP tool, ever. Next to it live `mappings.toml` (field mappings, aliases) and `knowledge.md` (free-form instance notes), both LLM-adjustable.
- **`mappings.toml`** — structured, whitelisted keys only: custom-field mappings (`story_points = "customfield_10002"`, `epic_link = "customfield_10008"`), board/project aliases (`"team alpha" = board 12`), default board/project, status-category overrides. Renderers consume these (e.g. points column in aggregate headers uses the mapped field).
- **`knowledge.md`** — free-form Markdown notebook for instance facts the model discovers or the user states: naming conventions, which board is which team, glossary, known JQL recipes for this org. Served back to the model at session start via MCP resource.
- **Self-discovery loop**: `jira_fields` lists all fields incl. custom (the API's field catalog); the agent finds "Story Points" → calls `config_set_mapping` → subsequent sprint reports have correct point sums. `atl init` seeds common mappings automatically by probing the field catalog.
- **Safety**: mapping writes validate against whitelisted key names and value shapes (field IDs must match `customfield_\d+` etc.); knowledge file is size-capped; neither file can contain secrets (scrubbed on write); the Atlassian **read-only guarantee is untouched** — these writes are local files only.

## 7. Functional scope — tools / subcommands

Every capability exists twice with the same name and semantics: MCP tool `jira_get_issue` ⇔ CLI `atl jira get PROJ-123`. **All tools are read-only toward Atlassian**; the only writes anywhere are the local-file adaptive-config tools (§6b).

### Jira (REST API v2, `/rest/api/2`)

| Capability | MCP tool | CLI |
|---|---|---|
| Get issue by key (fields, status, assignee, issue links, **remote/web links incl. Confluence pages**, subtasks, comments summary) | `jira_get_issue` | `atl jira get KEY` |
| Search by JQL (paginated, capped, summarized rows, **aggregate header**) | `jira_search` | `atl jira search "JQL"` |
| Issue changelog/history: status changes with timestamps, assignee changes, sprint moves ("when did it move to In Progress?", "how long blocked?") | `jira_issue_changelog` | `atl jira history KEY` |
| List comments of an issue | `jira_comments` | `atl jira comments KEY` |
| List available transitions of an issue (read-only workflow info) | `jira_transitions` | `atl jira transitions KEY` |
| List projects / issue types (discovery for JQL building) | `jira_projects` | `atl jira projects` |
| List versions/releases of a project (names, release dates, status) | `jira_versions` | `atl jira versions KEY` |
| Release report: all issues of a fixVersion with full rollup (done/remaining counts + points) — "what's left for 2.4?" | `jira_release_report` | `atl jira release KEY 2.4` |
| List saved filters visible to the user | `jira_filters` | `atl jira filters` |
| Current user (auth smoke test) | `jira_myself` | `atl jira whoami` |
| List all fields incl. custom fields (discovery for mappings, §6b) | `jira_fields` | `atl jira fields` |

### Jira Agile — boards & sprints (REST, `/rest/agile/1.0`)

Primary surface for BAs/POs analyzing sprint data for planning.

| Capability | MCP tool | CLI |
|---|---|---|
| List boards (filter by project/name) | `jira_boards` | `atl jira boards [--project KEY]` |
| List sprints of a board (active/future/closed) | `jira_sprints` | `atl jira sprints BOARD [--state active]` |
| Sprint contents + status/assignee/points breakdown, **with scope-change section: committed at start vs added mid-sprint vs removed/punted** (parity with Jira's own sprint report) | `jira_sprint_issues` | `atl jira sprint SPRINT_ID` |
| Board backlog (paginated, JQL-filterable, aggregate header) | `jira_backlog` | `atl jira backlog BOARD` |
| List epics of a board; epic progress rollup (children by status, points done/total) — "how done is epic PROJ-100?" | `jira_epics`, `jira_epic_issues` | `atl jira epics BOARD`, `atl jira epic KEY` |
| Velocity: committed vs completed points for the last N closed sprints of a board | `jira_velocity` | `atl jira velocity BOARD [--sprints 5]` |

### Confluence (REST, `/rest/api`)

| Capability | MCP tool | CLI |
|---|---|---|
| Get page by ID or by space+title (body converted to Markdown; **Jira issue macros render as issue keys + links, never dropped**) | `confluence_get_page` | `atl confluence get <id \| SPACE "Title">` |
| Extract all Jira issues referenced on a page (macros, links) with their current status — requirement traceability: "which stories cover this spec page?" | `confluence_page_issues` | `atl confluence issues <id>` |
| Search via CQL or plain text | `confluence_search` | `atl confluence search "query"` |
| List child pages / page tree | `confluence_children` | `atl confluence children <id>` |
| List spaces | `confluence_spaces` | `atl confluence spaces` |

### Cross-cutting

- `atl mcp` — run stdio MCP server (the command Copilot config points at).
- `atl mcp-config` — print ready-to-paste MCP client JSON for VS Code / Copilot CLI.
- `atl init` — interactive setup wizard for non-technical users: prompts for product base URLs and auth (PAT or user+pass, input masked), verifies each against its whoami endpoint live, writes the config file to the platform path, then prints the ready-to-paste MCP snippet with the absolute binary path filled in — with per-client instructions (VS Code `mcp.json` location vs Copilot app registration vs Copilot CLI), not just the JSON. Re-runnable (updates existing config). The intended one-and-only terminal touchpoint for a BA/PO. Two friction points the wizard must handle itself, not defer to config-file editing:
  - **PAT acquisition**: prints the exact per-product URL (`<base>/secure/ViewProfile.jspa` → Personal Access Tokens) and tells the user what to click before asking for the token.
  - **Corporate CA / TLS failure**: on a TLS verification error during live check, offers to accept a `ca_bundle` path (or explicit, warned, `insecure_skip_verify`) interactively and writes it to config — never dumps the user into hand-editing TOML.
- `atl check` — validate config, hit `myself`/whoami endpoints of each configured product, report reachability.

### Adaptive config (§6b) — the only tools that write anything, and only to local files

| Capability | MCP tool | CLI |
|---|---|---|
| Read current mappings + their sources | `config_get_mappings` | `atl config mappings` |
| Set/update a whitelisted mapping (validated) | `config_set_mapping` | `atl config set story_points customfield_10002` |
| Read instance knowledge notes | `knowledge_get` | `atl knowledge` |
| Append/update instance knowledge (size-capped, secret-scrubbed) | `knowledge_update` | `atl knowledge edit` |

## 8. Output structuring (the differentiator)

- Default output: **compact Markdown** designed for LLM consumption — stable heading structure, key facts first, bodies (descriptions, comments) rendered from Jira wiki-markup / Confluence storage-format HTML to Markdown.
- `--json` (CLI) / `format: "json"` (MCP arg): normalized JSON (our schema, not raw Atlassian) for scripting.
- **Aggregate header on every multi-issue tool** (`jira_search`, `jira_backlog`, `jira_sprint_issues`, `jira_epic_issues`, `jira_release_report`): true total count (from API, not rows shown), per-status counts, story-point sums. Rows may be capped; the numbers never are — stakeholder summaries scoped to a release/epic/quarter must be correct even when the result set exceeds the row cap, so the model never has to count rows itself.
- Hard caps everywhere: search results default 25 / max 100 rows; comment and page bodies truncated with `… (N more chars)` markers. Truncation is always explicit in output, never silent.
- Every entity includes its web URL so users can click through.

## 9. Agent-facing documentation (deliverable, not an afterthought)

The product ships a set of Markdown files written **for AI agents** (Copilot and other LLM clients), maintained in `docs/agents/` and included in every release archive. Purpose: today they make any MCP client use the tool well; later they become the instruction payload when `atl` is packaged as a **GitHub Copilot Extension / plugin for the Copilot marketplace**.

Files:

- `AGENTS.md` — top-level: what the tool is, read-only-toward-Atlassian guarantee, when to use which tool, tool-selection decision guide (e.g. "issue key mentioned → `jira_get_issue`; vague ask → `jira_search` with JQL"), pagination/truncation conventions, error meanings, and the **self-improvement loop**: when point sums look wrong or a board alias is unknown, discover via `jira_fields`, persist via `config_set_mapping` / `knowledge_update`, retry.
- `docs/agents/jira.md` — per-tool contract: arguments, defaults, output shape (with a real sample), JQL cookbook of proven queries for common asks ("open bugs assigned to me", "what didn't finish in sprint N", "everything updated this week in project X").
- `docs/agents/agile.md` — boards/sprints/backlog: how to resolve "the current sprint on board X" (boards → sprints state=active → sprint_issues), how to read the aggregate header, sprint-planning question recipes for BAs/POs.
- `docs/agents/confluence.md` — CQL cookbook, page-tree navigation patterns, storage-format caveats.

Rules:

- Every doc example is a **real, tested invocation** — golden tests render tool output into the docs' sample blocks so samples never drift from actual output.
- MCP tool descriptions in the server are generated from / kept consistent with these files (single source of truth; a CI check fails on drift).
- Docs are also exposed at runtime: MCP **resources** (`atl://docs/...`) so clients can pull usage guidance in-band, and `atl docs [topic]` on the CLI.
- Written for a model reader: imperative, example-first, no marketing prose, stable heading anchors.

Marketplace note: publishing the Copilot Extension itself (manifest, OAuth app, listing) is **out of scope for v1** — but these docs, stable tool names, and stable output schemas are the contract that packaging step will consume, so they are treated as public API from M1 on.

## 10. Error handling

- 401/403 → actionable message naming instance + auth mode ("PAT for jira.internal… rejected"), without leaking the credential.
- 404 → distinguish "issue does not exist" vs "no permission" where the API allows.
- Network/TLS errors surfaced with the target host and a hint (`ca_bundle`, `insecure_skip_verify`).
- MCP tool errors returned as tool-result errors (model-readable), never as protocol crashes.

## 11. Technical shape

- Go ≥ 1.22, module `github.com/bartekrozycki/atl-ai` (repo: https://github.com/bartekrozycki/atl-ai); binary name stays `atl`.
- MCP: official `modelcontextprotocol/go-sdk`, stdio transport only (v1).
- **Supported APIs only**: Jira `/rest/api/2`, Jira Agile `/rest/agile/1.0`, Confluence `/rest/api`. No legacy/internal endpoints (e.g. GreenHopper `/rest/greenhopper/...`) — ever. Derived metrics are computed client-side from supported data: `jira_velocity` from closed sprints + their issues' points; sprint scope-change (added/removed after start) from issue changelogs (`?expand=changelog` sprint-field entries). Slower but stable across DC versions.
- CLI: `spf13/cobra`.
- Layering: `internal/jira|confluence` typed clients → `internal/render` (Markdown/JSON) → thin `cmd/` (cobra) and `internal/mcpserver` frontends. Frontends contain no business logic.
- HTTP: stdlib `net/http`, shared client with timeout (default 30 s), retry-once on 429/503 with backoff.
- Tests: unit tests against recorded/stubbed HTTP fixtures (`httptest`); no live-instance dependency in CI. Golden-file tests for Markdown rendering.
- Release: GoReleaser → GitHub Releases with per-platform archives + checksums; Windows signing via SignPath Foundation, macOS sign+notarize via quill.
- **Public OSS**: MIT license, public GitHub repo (a SignPath Foundation eligibility requirement, and the Copilot-marketplace path assumes public docs/schemas anyway).

## 12. Milestones

Agent docs (§9) grow with each milestone: every tool lands together with its `docs/agents/` entry and golden-tested sample — docs are part of a tool's definition of done, not a trailing task.

1. **M1 — Skeleton + Jira read**: config/auth, `jira_get_issue`, `jira_search`, `whoami`, CLI + MCP frontends wired, `atl check`; `AGENTS.md` + `docs/agents/jira.md` started. *Usable end-to-end.*
2. **M2 — Agile + reporting read**: boards, sprints, sprint report with aggregates + scope changes, backlog, epics rollup, velocity, versions/release report, changelog; **adaptive config (§6b): `jira_fields`, mappings + knowledge tools, `atl init` mapping auto-seed** (point sums need the story-points mapping); `docs/agents/agile.md`. *BA/PO use-cases covered.*
3. **M3 — Confluence**: all remaining tools of §7 incl. `confluence_page_issues` traceability; `docs/agents/confluence.md`.
4. **M4 — Distribution polish**: GoReleaser pipeline, `atl init` wizard (incl. PAT guidance + interactive TLS/`ca_bundle` flow), `mcp-config` helper, MCP doc resources + `atl docs`, docs-drift CI check, README install docs for Copilot app / VS Code / Copilot CLI. The non-developer setup walkthrough must cover, screenshot-level: creating a PAT in Jira/Confluence DC, where to paste the MCP snippet per client, and what to do on a corporate-CA TLS error.

## 13. Acceptance criteria (v1 = M1–M4)

- Fresh machine, no runtimes: download binary → run `atl init` (URL + PAT, verified live) → paste printed MCP snippet into VS Code → Copilot answers "summarize PROJ-1234" correctly. No hand-editing of files required.
- A PO can ask Copilot "what's the status of the current sprint on board X?" and get the aggregate breakdown + issue list, and "what didn't finish last sprint?" and get the carry-over list.
- A PO can ask "what's left for release 2.4?" and "velocity of board X over the last 5 sprints?" and get correct rollup numbers (true totals, not capped-row counts).
- A BA can ask "list the Jira stories referenced on Confluence page 'Payments v2' with their statuses" and get every macro/link-referenced issue — no silently dropped references.
- On a result set larger than the row cap, the aggregate header still reports the true total; the model never fabricates counts.
- `atl jira get KEY` on an issue with 50+ comments produces output < ~8 KB by default.
- Wrong PAT produces a clear auth error naming the instance, not a stack trace.
- No code path issues a non-GET request to any Atlassian API (enforced by a unit test on the shared HTTP client).
- Adaptive-config writes cannot touch `config.toml`, reject non-whitelisted keys/shapes, and scrub secret-like strings (unit-tested); on an instance with story points in a custom field, the agent can discover + persist the mapping and the next sprint report shows correct point sums.
- Every MCP tool has a `docs/agents/` entry whose sample output is golden-tested against the real renderer; CI fails on tool↔docs drift.
- All unit tests pass offline.

## 14. Open questions

- OQ1: ~~GitHub org/module path~~ **Decided**: `github.com/bartekrozycki/atl-ai`, public.
- OQ2: ~~Code signing~~ **Decided**: repo is public OSS. Windows: **SignPath Foundation** (free OSS signing, GitHub Actions pipeline; apply for eligibility during M4). macOS: **Apple Developer Program** ($99/yr) → Developer ID cert + notarization, automated in CI via GoReleaser + quill (no Mac runner needed). Unsigned is not viable — enterprise endpoint policy blocks it (persona finding).
