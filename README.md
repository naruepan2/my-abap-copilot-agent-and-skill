# my-abap-copilot-agent-and-skill

Self-authored VS Code Copilot **agents** and **skills** for SAP/ABAP work, collected from the
author's `~/.copilot/agents` and `~/.copilot/skills` folders.

Only files written by the repo owner are kept here. Third-party / vendor-installed material
(agents and skills installed by **SAP AI Dev Toolkit**, plugins from other GitHub repositories,
etc.) is deliberately excluded — install those through their own installers instead.

## Layout

```
agents/   *.agent.md                                 (4)  — by Pongg
skills/   <name>/SKILL.md (+ supporting files)        (7)  — self-authored
```

Origin was derived from `~/.copilot/roster.json` (agents) and
`~/.copilot/.sap-ai-dev-toolkit-assets.json` (SHA-256-tracked toolkit files). Files installed by
SAP AI Dev Toolkit were removed — use the toolkit upgrade to get them.

Agents and skills are plain Markdown with YAML frontmatter. New files load in a **new chat
session**, not in the one where they were added.

## Agents (4) — by Pongg

| File | Purpose |
|---|---|
| `abap-developer-by-pongg.agent.md` | Coordinator for ABAP work; delegates read-only tasks using `~/.copilot/roster.json` allowlist (default-deny). |
| `sap-abap-reverse-engineer-by-pongg.agent.md` | Strictly read-only, evidence-driven ABAP/CDS/RAP reverse engineering. |
| `sap-se38-report-tester-by-pongg.agent.md` | SE38 report testing without SAP GUI, selection-screen replay, bug hunting. |
| `sap-st22-dump-analysis-by-pongg.agent.md` | Read-only ST22 dump analysis via ARC-1 SAPDiagnose. |

These four are the three `delegatable` entries (reverse engineer, SE38 tester, ST22 analyst) plus
the coordinator itself in `~/.copilot/roster.json`. Hard rule: delegate read-only work only — never
create, update, delete, activate, publish, or transport.

## Skills (7) — self-authored

| Skill | Purpose |
|---|---|
| `arc-1-add-sap-system` | Register a new SAP logon in ARC-1 MCP configuration (VS Code `mcp.json` + Copilot CLI `mcp-config.json`). |
| `capture-lesson` | Turn a session finding into a durable artifact by choosing the right destination and waiting for approval. |
| `long-task-control` | Checkpoint and stop work that outlives one request; stop condition, handoff, evidence expiry. |
| `mcp-surface-audit` | Rebuild the real SAP/ABAP MCP tool surface from live config and manifests, then fix routing drift. |
| `new-program` | Scaffold local project/program records — `project.yaml`, README, `program.yaml`, spec index — without guessing SID/client/transport. |
| `rap-lifecycle` | The ordered RAP create-and-publish chain: which server owns each step and which checks stop a half-published service. |
| `sap-adam-external-debug` | External ABAP debugging with the ADAM MCP server: listener conflicts, `/h` vs external debugger, SM51 runs, timeouts, interactive sessions, and verified cleanup. Program-agnostic. |

## Installation

Copy the folders into your VS Code user profile:

```powershell
Copy-Item .\agents\*  "$env:APPDATA\Code\User\agents" -Force
Copy-Item .\skills\*   "$env:APPDATA\Code\User\skills" -Recurse -Force
```

Adjust the destination if your editor or Copilot extension uses a different customization folder
(e.g. `~/.copilot/agents` and `~/.copilot/skills`). Restart VS Code / start a new chat session to
load the files.

Several files expect configured SAP MCP servers (ARC-1, ADAM, SAP AI Dev Toolkit) with an
authorized SAP destination. No credentials are stored in this repository.

## Notes

- Program-agnostic — no file is tied to a specific Z program or customer repository.
- System IDs, clients, and user names in examples are illustrative; review them before use.
- To get the vendor agents/skills, use their installers (SAP AI Dev Toolkit upgrade,
  `npx skills add ...`, plugin marketplace). They are intentionally not duplicated here.
