# Backlog — `Atl` PowerShell module

Derived from PRD §12 milestones. One task = one PR-sized unit. Every cmdlet task includes: comment-based help (when-to-use / when-NOT-to-use + example), `docs/agents/` entry with golden-tested sample, Pester tests against fixtures. That's the definition of done (PRD §9, §12) — not repeated per task below.

Status: `[ ]` todo · `[~]` in progress · `[x]` done

---

## M1 — Skeleton + Jira read (usable end-to-end)

### Foundation
- [ ] **M1-01** Repo scaffold: `Atl/` module layout (`Atl.psd1`, `Atl.psm1`, `Public/`, `Private/`), `Tests/`, `PSScriptAnalyzerSettings.psd1`, MIT `LICENSE`, `README.md` stub.
- [ ] **M1-02** CI workflow: Pester 5 + PSScriptAnalyzer on push/PR (`pwsh` on ubuntu + windows runners).
- [ ] **M1-03** `Private/Invoke-AtlRequest.ps1` — the choke point: GET-only hard reject, auth header injection (PAT Bearer / Basic), `User-Agent: Atl/<version> (+repo)`, timeout 30 s, `Retry-After` honor, jittered backoff, TLS options (OS trust store, `ca_bundle`, config-only `insecure_skip_verify` with warning). Pester: non-GET throws; retry behavior; UA present; **no request to any non-configured host** (no-telemetry test).
- [ ] **M1-04** `Private/Config.ps1` — `config.json` load, parameter/env/file precedence, per-OS paths, ACL enforcement (chmod 600 / icacls), secrets never in errors/logs. Pester: precedence matrix, ACL set, secret-leak greps on error output.
- [ ] **M1-05** `Private/Errors.ps1` — ErrorRecord categories (auth / no-permission / not-found / network / parse), exit codes for `pwsh -Command`, 401/403 message naming instance + auth mode.

### Jira core
- [ ] **M1-06** `Private/JiraService.ps1` (part 1) + `Get-AtlJiraIssue` — fields, status, assignee, issue links, remote/web links, subtasks, comments summary; `-Include comments,changelog,transitions`. Depends: M1-03..05.
- [ ] **M1-07** `Private/Render.Markdown.ps1` (core) — issue → compact MD, stable headings, truncation markers, web URLs, < 8 KB on 50+ comments (acceptance §13). Golden tests.
- [ ] **M1-08** `Private/Render.JiraWiki.ps1` — Jira wiki-markup → Markdown (bold/lists/links/code/quotes/tables; unknown markup passes through legibly). Golden tests.
- [ ] **M1-09** `Search-AtlJira` — JQL, pagination (`-Start`/`-Limit`, default 25/max 100), summarized rows, **aggregate header with true totals**, in-band continuation line, JQL parse-error relay with offending clause. Depends: M1-06.
- [ ] **M1-10** `Get-AtlJiraMeta` — kinds: `projects | fields | filters | whoami` (boards/sprints/versions/epics land in M2). `whoami` = auth smoke test.
- [ ] **M1-11** `Test-AtlConnection` — config validation + whoami per configured product, reachability report, actionable TLS/auth hints.
- [ ] **M1-12** `-AsObject` / `-Json` output modes on all M1 cmdlets (normalized schema, documented as public API).
- [ ] **M1-13** `AGENTS.md` v1 + `docs/agents/jira.md` — decision guide, JQL cookbook (≥ 10 proven queries), pagination/truncation conventions, error meanings, self-improvement loop stub.
- [ ] **M1-14** Docs-drift CI check v1 — golden samples in docs rendered from fixtures; fail on drift.

**M1 exit test**: fresh machine, manual module import → `Test-AtlConnection` green → Copilot agent mode answers "summarize PROJ-1234" via `Get-AtlJiraIssue`.

---

## M2 — Agile + reporting read (BA/PO covered)

### Agile plumbing
- [ ] **M2-01** `Get-AtlJiraMeta` kinds `boards | sprints | versions | epics` (+ scope params).
- [ ] **M2-02** `Private/Resolve.ps1` — fuzzy board/sprint/epic/project resolution: name match + `mappings.json` aliases + IDs; ambiguity error enumerating candidates with IDs. Pester: exact, fuzzy, alias, ambiguous, none.
- [ ] **M2-03** Changelog fetch helper — `?expand=changelog` bulk crawl, bounded `ForEach-Object -Parallel` ≤ 4, sprint/issue caps (PRD §11).

### Reporting cmdlets
- [ ] **M2-04** `Get-AtlSprintReport` — one-call `BOARD current|last|name|id`; aggregate header (total, per-status, points done/remaining, unassigned); **scope-change section** (committed vs added mid-sprint vs punted) from changelog sprint-field entries. Depends: M2-02, M2-03. *Riskiest task in backlog — spike first on a real DC instance.*
- [ ] **M2-05** `Search-AtlJira -Board X -Backlog` — board-scoped backlog mode with aggregate header.
- [ ] **M2-06** `Get-AtlEpicReport` — children by status, points done/total.
- [ ] **M2-07** `Get-AtlReleaseReport` — fixVersion rollup, done/remaining counts + points, true totals beyond row cap.
- [ ] **M2-08** `Get-AtlVelocity` — last N closed sprints, committed vs completed points (committed = points at sprint start, via changelog). Depends: M2-03, M2-04 machinery.

### Adaptive config (§6b)
- [ ] **M2-09** `Private/Mappings.ps1` + `Set-AtlMapping` — whitelist (story_points, epic_link, flagged, aliases, defaults, status-category overrides), shape validation (`customfield_\d+`), provenance stamps (when/who). Pester: reject non-whitelisted, reject bad shapes, config.json untouchable.
- [ ] **M2-10** `Private/Knowledge.ps1` + `Add-AtlKnowledge` — `ShouldProcess`/`-Confirm` gate, size cap, secret scrub, timestamps. Pester: no write without confirm, cap, scrub.
- [ ] **M2-11** `Get-AtlContext` — mappings + knowledge in one call, untrusted-notes framing wrapper.
- [ ] **M2-12** Provenance in renderer — mapped numbers cite source mapping + set-date in aggregate headers. Depends: M2-09.
- [ ] **M2-13** `docs/agents/agile.md` — one-liner recipes (current sprint, carry-over, release status, velocity), aggregate-header reading guide, BA/PO question recipes, self-improvement loop full text.

**M2 exit test**: PO asks "status of current sprint on board X" → one command, correct points with provenance; "what's left for 2.4" + "velocity last 5 sprints" correct on >100-issue sets.

---

## M3 — Confluence

- [ ] **M3-01** `Private/ConfluenceService.ps1` — page by id / space+title, children, spaces, CQL/text search.
- [ ] **M3-02** `Private/Render.StorageFormat.ps1` — storage-format HTML → Markdown; **Jira issue macros render as keys + links, never dropped** (acceptance §13). Golden tests incl. macro, table, layout, code-block fixtures. *Second-riskiest task — budget 2× estimate.*
- [ ] **M3-03** `Get-AtlConfluencePage` — `-Include issues` returns referenced Jira issues (macros + links) with current status (traceability, G2c). Depends: M3-01, M3-02.
- [ ] **M3-04** `Search-AtlConfluence` — CQL or plain text, aggregate header, CQL parse-error relay.
- [ ] **M3-05** `Get-AtlConfluenceChildren`, `Get-AtlConfluenceSpaces`.
- [ ] **M3-06** `docs/agents/confluence.md` — CQL cookbook, page-tree patterns, storage-format caveats.

**M3 exit test**: BA asks "stories referenced on page 'Payments v2' with statuses" → complete list, zero dropped references.

---

## M4 — Distribution polish

- [ ] **M4-01** `Initialize-Atl` wizard — URL + auth prompts (masked), live whoami verify, PAT-acquisition guidance (per-product profile URL), interactive `ca_bundle` flow on TLS failure (never suggests skip-verify), mapping auto-seed via field-catalog probe, writes `config.json` + ACL, re-runnable.
- [ ] **M4-02** `Get-AtlAgentSetup` — `copilot-instructions.md` snippet + per-client placement instructions (VS Code / Copilot CLI / Copilot app).
- [ ] **M4-03** `Get-AtlDocs [topic]` — print embedded docs at runtime.
- [ ] **M4-04** Release pipeline — version bump, `Publish-Module` to PowerShell Gallery from CI, GitHub release zip.
- [ ] **M4-05** SignPath Foundation application + Authenticode signing wired into release. *External dependency — apply at M4 start, not end.*
- [ ] **M4-06** README — install (Gallery + offline zip), setup walkthrough screenshot-level: PAT creation in Jira/Confluence DC, agent-mode enable + snippet placement per client, corporate-CA TLS remediation.
- [ ] **M4-07** Acceptance sweep — walk every §13 criterion, fix stragglers, tag v1.0.0.

**M4 exit test**: PRD §13 first bullet verbatim on a fresh Windows machine.

---

## v1.1 (not scheduled — PRD §12b, persona-ranked)

`Get-AtlFlowReport` · standup-delta tool · multi-board rollups · link-graph traversal · text attachments · watchers/worklog · argument completers/`-All`/`-Raw`/`Open-AtlIssue` · SecretManagement-required mode · SBOM/provenance.
