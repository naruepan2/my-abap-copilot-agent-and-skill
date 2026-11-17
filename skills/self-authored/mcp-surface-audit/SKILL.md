---
name: mcp-surface-audit
description: Rebuild the real SAP/ABAP MCP tool surface from live config and extension manifests, then fix routing drift. Use when a new MCP server or ABAP extension is installed, when the agent picks the wrong server for a task, when a tool name referenced by an agent or skill no longer resolves, or when asked to "audit MCP tools", "check for duplicate MCP tools", "why did it use the wrong tool", "which server should do X", or "refresh the tool routing".
---

# MCP surface audit

The tool surface changes whenever a server, plugin, or ABAP extension updates. Never answer from
memory or from a project README: those lag. Regenerate the surface first, then patch the one place
that holds routing.

Rules that make this audit trustworthy:

- Extension manifests and `tools/list` are authoritative. A README is not.
- Extension-contributed tools and MCP-server tools both consume context. Count both.
- Routing lives in exactly one artifact. Duplicating it creates a second, silently-stale truth.

## 1. Regenerate the surface

Run these in a terminal with `mode=sync`. They are read-only and fast. **Run each as a single line** —
the integrated terminal truncates multi-line PowerShell pastes.

**a. Configured MCP servers and their SAP/ARC-1 env keys**

```powershell
$m = Get-Content "$env:APPDATA\Code\User\mcp.json" -Raw | ConvertFrom-Json; $m.servers.PSObject.Properties | ForEach-Object { [pscustomobject]@{ Server=$_.Name; Type=$_.Value.type; Command=$_.Value.command; SapEnv=(($_.Value.env.PSObject.Properties.Name | Where-Object { $_ -match '^(SAP_|ARC1_)' }) -join ',') } } | Format-Table -AutoSize
```

**b. Extension-contributed tool counts (invisible in `mcp.json`)**

```powershell
Get-ChildItem "$env:USERPROFILE\.vscode\extensions" -Directory | ForEach-Object { $p = Join-Path $_.FullName 'package.json'; if (Test-Path $p) { $j = Get-Content $p -Raw -ErrorAction SilentlyContinue | ConvertFrom-Json -ErrorAction SilentlyContinue; if ($j -and $j.contributes) { $lm = @($j.contributes.languageModelTools) | Where-Object { $_ }; $prov = $null -ne $j.contributes.mcpServerDefinitionProviders; if ($lm.Count -gt 0 -or $prov) { [pscustomobject]@{ Extension=$_.Name; LMTools=$lm.Count; McpProvider=$prov } } } } } | Sort-Object LMTools -Descending | Format-Table -AutoSize
```

**c. Exact tool names for one extension**

```powershell
$p = "$env:USERPROFILE\.vscode\extensions\<extension-folder>\package.json"; (Get-Content $p -Raw | ConvertFrom-Json).contributes.languageModelTools | ForEach-Object { $_.name } | Sort-Object
```

**d. Per-system authorization for the SAP official ADT MCP server**

Its `destinations` allowlist decides which systems it can reach at all. Missing a system here means
the server cannot be the primary tool for that system, regardless of feature comparisons.

```powershell
$s = Get-Content "$env:APPDATA\Code\User\settings.json" -Raw; if ($s -match '"adt\.mcpServer\.accessRestrictions"\s*:\s*\{[^}]*\}') { $Matches[0] }
```

**e. Live tool names for each running MCP server**

Open **MCP: List Servers**, start the server, then expand it in the Chat tools picker, or read the
server's Output channel. `tools/list` is the only authoritative name source — use the exact strings
it returns, including any destination prefix.

**f. Tool names currently referenced by agents and skills**

Any name this prints that is absent from (e) is drift.

```powershell
Get-ChildItem "$env:USERPROFILE\.copilot\agents","$env:USERPROFILE\.copilot\skills","$env:USERPROFILE\.agents\skills" -Recurse -Include *.md | Select-String -Pattern 'SAP(Read|Write|Activate|Search|Context|Diagnose|Lint|Transport|Git|Manage|Query|Navigate)\b|abapfs_[a-z0-9_]+|abap_[a-z0-9_]+' -AllMatches | ForEach-Object { $_.Matches.Value } | Sort-Object -Unique
```

**g. Skill inventory — there are TWO user roots, plus extension-provided skills**

Skills load from both roots, and count toward context just like tools. Count both; auditing only one
understates the surface.

```powershell
@("$env:USERPROFILE\.agents\skills","$env:USERPROFILE\.copilot\skills") | ForEach-Object { $c = (Get-ChildItem $_ -Directory -ErrorAction SilentlyContinue | Where-Object { Test-Path (Join-Path $_.FullName 'SKILL.md') }).Count; "$_ => $c skills" }
```

Which skills any updater can actually refresh — a lock entry is not proof of installation, and an
installed skill is not proof of a lock entry:

```powershell
$l = (Get-Content "$env:USERPROFILE\.agents\.skill-lock.json" -Raw | ConvertFrom-Json).skills.PSObject.Properties.Name; "lock entries: $($l.Count)"; "installed in .agents: $((Get-ChildItem "$env:USERPROFILE\.agents\skills" -Directory | Where-Object { Test-Path (Join-Path $_.FullName 'SKILL.md') }).Count)"; "tracked and installed: $(($l | Where-Object { Test-Path (Join-Path "$env:USERPROFILE\.agents\skills" $_) }).Count)"
```

Skills in `~/.copilot/skills` written by an installer (for example SAP AI Dev Toolkit) are updated by
reinstalling that installer's package, not by `npx skills update`. Skills placed there by hand are
updated by editing them.

## 2. Interpret before writing anything

Estimate context cost at roughly 250 tokens per advertised tool. Report the total next to the
count, because the count is the thing that actually degrades tool selection.

For every capability that now appears on two or more servers, decide with these criteria, in order:

1. **Can it reach the target system?** Authorization and connectivity beat features. A server that
   cannot log on to the system is never the answer for that system.
2. **Does it actually have the operation?** Confirm from `tools/list`, not from a capability essay.
   Absence of a specific action (batch activate, transport release, publish, grep, V4 binding) is
   common and decides the routing more often than general breadth does.
3. **Is the write path safe?** Prefer the server that validates, guards with a source hash, or
   batches activation in dependency order.
4. **Is the guidance authoritative?** Prefer system-derived generators over hardcoded templates, and
   real ATC over local heuristics.
5. Only then prefer fewer tools / lower token cost.

Record the reason, not just the winner. A bare "use X" ages badly; "use X because Y lacks Z" stays
useful after an upgrade.

## 3. Write findings to exactly one place

| Finding | Destination |
| --- | --- |
| Task → tool routing table | The task's agent file, under a routing section |
| Per-server system reachability | A short pointer note in memory, dated |
| The audit procedure itself | This skill |

Never copy the routing table into memory, a second agent, or `AGENTS.md`. `AGENTS.md` may state
principles only ("one task, one server"; "if the first choice is denied, report it instead of
quietly falling back").

When patching an agent, keep the table to task rows, name the first choice, and name what must not
be used. Add a dated line stating when the reachability was verified.

## 4. Verify the patch

- Every tool name written into an agent or skill must exist in (e). No PascalCase guesses.
- Frontmatter `tools:` prefixes must match the server names from (a) and the extension names from
  (b); a renamed server silently invalidates an allowlist.
- Reload the window or start a new chat, then confirm the agent appears and its tools bind.

## Tool entry matching — verified rule

`tools:` entries match a tool by prefix. The workbench picker computes checked state as:

```js
checked: allChecked || entries.includes(toolId)
      || entries.some(h => h.endsWith("/*") && toolId.startsWith(h.slice(0, -1)))
```

`h.slice(0, -1)` drops the trailing `*`, so an entry `"<prefix>/*"` matches every tool id
beginning with `"<prefix>/"`. Consequences:

- A wildcard is a **prefix** match, not a glob. `a/*` never matches `ab/...`.
- The prefix must equal the **client tool id** prefix exactly, including case and spaces.
  Internal runtime registration names are not it: an extension tool's id is
  `<extensionId>/<toolName>` (`murbani.vscode-abap-remote-fs/abapfs_...`), not the
  `abapfs_...` runtime name.
- MCP tools group by the text before the first `/`. The picker displays that group label,
  which makes it the fastest way to read back the real prefix.

### Verified against a live agent, 2026-11-13

Probed by spawning the agent and having it report its own tool list:

| Entry | Result |
| --- | --- |
| `arc-1/*` | ✅ resolves — the key declared in `mcp.json` |
| `adt-mcp-server/*` | ✅ resolves — even though the server's display label is "ADT MCP Server" |
| `murbani.vscode-abap-remote-fs/<toolName>` | ✅ resolves, one entry per tool |
| `ADT MCP Server/*` | ❌ display label does not match |
| `sapse.adt-vscode/*` | ❌ owning extension id does not match |
| `com.sap.adt/*` | ❌ provider id does not match |
| `murbani.vscode-abap-remote-fs/*` | ❌ wildcard does not expand for extension tools |

Two rules follow. **Match an MCP server by its `mcp.json` key**, never by its label or its
runtime tool id (`mcp_arc-12_*`, `mcp_adt_mcp_serve_*`). **Give every extension-contributed
tool its own explicit entry**; a wildcard there yields zero tools and no error.

### Self-reports are samples, not inventories

The same agent reported 10, 9 and 8 ARC-1 tools, and 13 then 6 ABAP FS tools, across
consecutive spawns. Verify **presence of a group** and **whether a named entry resolves**.
Do not delete a working line because one probe under-reported it, and do not publish counts
as facts.

### When an entry stops resolving

Do not widen it to a wildcard. Either name the exact tool from the extension's
`languageModelTools` manifest, or drop the `tools:` list entirely — an omitted list lets the
session tools picker decide, which is the safe fallback.

The 13 ABAP Remote FS entries are written out in `tools:` because a wildcard does not expand
for extension-contributed tools (verified above). Ask the agent for a count rather than
trusting a one-time listing, for the reason in the previous section.

## Agent roster drift

`~/.copilot/roster.json` is the delegation allowlist. Drift means the installed agents and
the roster disagree. Run this as one line:

```powershell
$j = Get-Content "$env:USERPROFILE\.copilot\roster.json" -Raw | ConvertFrom-Json; $known = @(); foreach ($d in $j.delegatable) { $known += $d.file }; foreach ($g in $j.not_delegatable) { $known += $g.files }; $live = Get-ChildItem "$env:USERPROFILE\.copilot\agents" -Filter *.agent.md | ForEach-Object { $_.Name -replace '\.agent\.md$','' }; $new = $live | Where-Object { $known -notcontains $_ }; $gone = $known | Where-Object { $live -notcontains $_ }; "NEW: $(if ($new.Count -gt 0) { $new -join ', ' } else { 'none' })"; "GONE: $(if ($gone.Count -gt 0) { $gone -join ', ' } else { 'none' })"
```

**Both lists must be counted.** Checking only `delegatable` reports every installer-owned agent
as new on every run, which trains the reader to ignore the warning. Expect `GONE: none` and
`NEW: none`; the actual totals are whatever `roster.json` and the agents folder hold this time —
read them from that output instead of quoting a number remembered from a previous run. A count
written into prose here goes stale exactly like any other duplicated fact.

**Run this manually.** Do not wait for the `sessionStart` drift hook: per-event firing differs in
this harness, and `chat.useHooks` is documented as Local-harness only while
`chat.defaultToCopilotHarness` defaults to `true`. Treat the drift check as an on-demand step, and
never report a hook as having run until a firing is observed in a log or a changed file. Measured
per-event behaviour: `docs/reference/environment.md`.

Interpretation:

| Output | Meaning | Action |
| --- | --- | --- |
| `NEW: <name>` | an agent appeared that the roster does not classify | report it; a human decides whether to add it. It is **not** delegatable meanwhile |
| `GONE: <name>` | the roster names an agent that no longer exists | report the dangling entry; do not substitute a similar agent |
| both `none` | in sync | — |

Do not add or remove roster entries yourself. The allowlist is only meaningful if a human
maintains it.

## Installer-owned files


Some files under `~/.copilot/agents` and `~/.copilot/skills` are installed by a package and
tracked by hash in `~/.copilot/.sap-ai-dev-toolkit-assets.json`. Editing one is allowed but
changes the contract: the installer will report it as "needs manual review" and stop
overwriting it, so that file no longer receives upstream updates. Before editing, check
ownership and say so:

```powershell
$j = Get-Content "$env:USERPROFILE\.copilot\.sap-ai-dev-toolkit-assets.json" -Raw | ConvertFrom-Json; $j.files.PSObject.Properties.Name | Where-Object { $_ -like 'agents*' }
```

Prefer creating a separate `*-by-<owner>.agent.md` over modifying an installed file, unless
the owner explicitly accepts losing that file's updates.

## Red flags observed in practice

- A memory note asserting a routing rule that is older than the last server upgrade. It was wrong on
  7 points after two months, including "activation: ADT MCP only" on a server that has no batch
  activate.
- A README-advertised tool count taken as fact. Published counts and real surfaces differed by 3x.
- Extension tools forgotten entirely because they never appear in `mcp.json`.
- A remembered security workaround (`SAP_INSECURE=true`) surviving long after the real fix (a CA
  bundle) landed. Re-verify any note that relaxes TLS or auth.
