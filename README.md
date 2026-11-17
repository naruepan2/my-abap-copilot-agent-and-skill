# my-abap-copilot-agent-and-skill

Personal VS Code Copilot **agents** and **skills** for SAP/ABAP work, collected from the author's
`~/.copilot/agents` and `~/.copilot/skills` folders.

## Layout — grouped by origin

```
agents/
  self-authored-by-pongg/   *.agent.md    (4)   — written by the repo owner
  sap-ai-dev-toolkit/       *.agent.md    (13)  — installed by SAP AI Dev Toolkit
skills/
  self-authored/            <name>/SKILL.md (7)  — written by the repo owner
  sap-ai-dev-toolkit/       <name>/SKILL.md (12) — installed by SAP AI Dev Toolkit
```

Origin is not guessed from names: it is derived from
`~/.copilot/roster.json` (agents; the toolkit entries are the `not_delegatable` ones installed by
the toolkit, the `by-pongg` entries are self-authored) and
`~/.copilot/.sap-ai-dev-toolkit-assets.json` (SHA-256-tracked toolkit agents and skills). Skills
absent from that manifest — and not present in the installed ARC-1 plugin — are self-authored.

Toolkit files are kept for reference (version tracked by the toolkit). Do not edit them in this
repo expecting VS Code to pick up the change; the toolkit re-installs its versions on upgrade.

Agents and skills are plain Markdown with YAML frontmatter. New files load in a **new chat session**.

## Agents — self-authored, by Pongg (4)

`agents/self-authored-by-pongg/`

| File | Purpose |
|---|---|
| `abap-developer-by-pongg.agent.md` | Coordinator for ABAP work; delegates read-only tasks using `~/.copilot/roster.json` allowlist. |
| `sap-abap-reverse-engineer-by-pongg.agent.md` | Strictly read-only, evidence-driven ABAP/CDS/RAP reverse engineering. |
| `sap-se38-report-tester-by-pongg.agent.md` | SE38 report testing without SAP GUI, selection-screen replay, bug hunting. |
| `sap-st22-dump-analysis-by-pongg.agent.md` | Read-only ST22 dump analysis (ARC-1 SAPDiagnose). |

## Agents — installed by SAP AI Dev Toolkit (13)

`agents/sap-ai-dev-toolkit/`

| File | Purpose |
|---|---|
| `abap-developer.agent.md` | ABAP/CDS/RAP implementation, debugging, validation, transport prep (SAP AI Dev Toolkit MCP). |
| `abap-runtime-debugger.agent.md` | Runtime incidents, logs, authorized debugger sessions, performance symptoms. |
| `hana-cloud-hdi-specialist.agent.md` | HANA Cloud / HDI containers, CAP HANA models, deployment handoffs. |
| `rap-service-developer.agent.md` | RAP business objects, behavior, projections, service definitions/bindings. |
| `sap-advanced-event-mesh-administrator.agent.md` | SAP Integration Suite, advanced event mesh (Solace) administration. |
| `sap-btp-infrastructure-admin.agent.md` | BTP account hierarchy, entitlements, service keys, destinations, trust, Cloud Connector. |
| `sap-bw-datasphere-migration-specialist.agent.md` | BW to Datasphere migration, reconciliation. |
| `sap-datasphere-developer.agent.md` | Datasphere modeling, loads, data products, SQL checks, administration. |
| `sap-devops-observability-specialist.agent.md` | Cloud ALM monitoring, incident triage, deployment tracking. |
| `sap-integration-architect.agent.md` | Integration Suite lifecycle design and plan handoff. |
| `sap-integration-developer.agent.md` | Executes the approved Integration Suite lifecycle. |
| `sap-process-automation-developer.agent.md` | SAP Build Process Automation instances, tasks, events, administration. |
| `sap-solution-architect.agent.md` | Standard-solution assessment, Clean Core design, SDLC orchestration. |

## Skills — self-authored (7)

`skills/self-authored/`

| Skill | Purpose |
|---|---|
| `arc-1-add-sap-system` | Register a new SAP logon in ARC-1 MCP configuration. |
| `capture-lesson` | Turn a session finding into a durable artifact with approval. |
| `long-task-control` | Checkpoint and stop work that outlives one request. |
| `mcp-surface-audit` | Rebuild the live SAP/ABAP MCP tool surface, fix routing drift. |
| `new-program` | Scaffold project/program records (project.yaml, README, spec index). |
| `rap-lifecycle` | Ordered RAP create-and-publish chain and publication checks. |
| `sap-adam-external-debug` | ADAM external debugger setup/recovery: listener conflicts, `/h` vs external, SM51 runs, cleanup. |

## Skills — installed by SAP AI Dev Toolkit (12)

`skills/sap-ai-dev-toolkit/`

| Skill | Purpose |
|---|---|
| `abap-debugging` | ABAP runtime failure investigation with logs, source inspection, safe reads. |
| `abap-development` | Implement and change ABAP programs, classes, function groups, reports. |
| `abap-runtime-analysis` | Runtime incidents, debugger state, performance symptoms. |
| `abap-testing-quality` | ABAP Unit tests, lint, diagnostics, syntax checks, ATC. |
| `cds-development` | ABAP CDS data definitions, dependencies, consumers. |
| `clean-core-extensibility` | Clean Core extensions, released extension points, side-by-side patterns. |
| `rap-development` | RAP business objects, behavior, projections, OData services. |
| `rap-service-delivery` | RAP OData validation, activation, publication, runtime checks. |
| `sap-sdlc-orchestration` | SAP delivery lifecycle from requirement to operational handover. |
| `sap-standard-api-analysis` | Assess standard capabilities and released APIs before custom code. |
| `sap-transport-release` | Transport readiness, request contents, tests, activation. |
| `sap-webgui-fallback` | Read-only SAP GUI for HTML transactions (jobs, dumps, auth checks, etc.). |

## Installation

Copy the folders into your VS Code user profile:

```powershell
# The grouped subfolders are flattened into the customization directories:
Get-ChildItem .\agents\*\*.agent.md | Copy-Item -Destination "$env:APPDATA\Code\User\agents" -Force
Get-ChildItem .\skills\*\* | Copy-Item -Destination "$env:APPDATA\Code\User\skills" -Recurse -Force
```

To install only the self-authored set, point the globs at `agents\self-authored-by-pongg\*` and
`skills\self-authored\*` instead.

Adjust the destination if your editor or Copilot extension uses a different customization folder
(e.g. `~/.copilot/agents` and `~/.copilot/skills`). Restart VS Code / start a new chat session to
load new files.

Most agents/skills expect configured SAP MCP servers (SAP AI Dev Toolkit / ARC-1 / ADAM /
sap-ai-*-servers) with an authorized SAP destination; the Markdown does not contain credentials.

## Notes

- Program-agnostic: none of the files is tied to a specific Z program or customer repository.
- Skills from `~/.agents/skills` (third-party/plugin families such as ARC-1, caveman, ponytail,
  mattpocock) are intentionally not included here — only the author's `.copilot` set.
- Paths, system IDs, and user names in examples are illustrative; review them before use.
