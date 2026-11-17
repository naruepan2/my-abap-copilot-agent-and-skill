---
name: sap-adam-external-debug
description: Set up and recover external ABAP debugging on an SAP system with the ADAM MCP server (abap_debug / abap_debug_session / abap_breakpoints / abap_source). Use when an ABAP debug session will not arm, reports a listener conflict or "another session has global debugging scope", stops only in the SAP GUI internal debugger, times out although the program ran, or when stepping/inspecting a held SAP program. Program-agnostic: works for any ABAP program, class or function module.
---

# SAP external ABAP debug (ADAM)

This skill covers only the **setup and recovery** of external debugging through the ADAM MCP
server (`mcp_abap-debugger_*`). It is not tied to any program, selection screen, or business process.
For diagnosing a runtime failure once the run is captured, use `abap-runtime-analysis` instead.

## Tool surface (live names)

Query the active MCP server's `tools/list` once and use the exact live names; ADAM exposes them
unprefixed when scoped to one destination:

- `abap_breakpoints` — `list` / `delete` (with `listener: true`)
- `abap_source` — active source with the line numbers breakpoints address
- `abap_debug` — arm breakpoints and capture a run; `interactive: true` holds the program
- `abap_debug_session` — `step` / `continue` / `inspect` / `watch` / `wait` / `end`

## Rule 1 — Preflight before every arm

Call `abap_breakpoints(action="list")` first. Interpret `listener`:

| `listener` value | Meaning | Action |
|---|---|---|
| `none` | clear | proceed |
| `your capture/session ...` | your own active capture | `abap_debug_session(action="end")` first |
| `another debugger: ...` or `Another session ... global debugging scope` | a debuggee/listener holds the SAP user | `abap_breakpoints(action="delete", listener:true)`, re-list; if it persists, the user must leave the debugger in SAP GUI or log off that SAP session |

`delete + listener:true` removes a **listener**, not a suspended debuggee. If SAP still reports
`global debugging scope`, a SAP GUI session is holding it — never kill the MCP process, ask the
user to press **Leave Debugger**, close the SAP GUI window, or log off/relogin.

## Rule 2 — Verify the breakpoint line

`abap_source(type, name, lineStart/lineEnd or grep)` must show an **executable statement** at
the chosen line. Blank lines and full-line comments (`*` or `"` in column 1) cannot carry a
breakpoint; SAP refuses them (the tool probes nearby lines and reports the ones tried).

For classic reports, local classes live in includes (`INCL`); there is no `CLAS` object — grep
the include source to find the method and line.

## Rule 3 — Arm correctly

```
abap_debug {
  type: "PROG" | "CLAS" | "FUNC", name: <object>, line: <N>,
  trigger: "none", runFrom: "sapgui", timeoutSeconds: 300-600,
  variables: [ ... ]        // name everything needed up front; a run is released before you can ask again
}
```

- Prefer `trigger:"unittest"` (harmless tests) or `trigger:"http"` (OData GET) when the code
  allows — they avoid the SAP GUI session problem entirely. Use `trigger:"none"` only for a
  report/screen or a save/action; HTTP trigger cannot perform writes.
- `interactive: false` (default) = one-shot: stop, record, release immediately; result has
  `stops`, `variables`, `release`.
- `interactive: true` = holds the program at the first stop for `abap_debug_session` moves.
  `interactive` changes **retention only**, not whether the breakpoint can be armed.
- Never supply extra variables you do not need; each adds cost per stop/step.

## Rule 4 — Run contract (the most common failure)

With `trigger:"none"` the user must start the program as the **same SAP user** while the capture
waits, inside `timeoutSeconds`:

1. Do **not** type `/h` — that opens the internal debugger, which MCP cannot see
   (`noSessionAttached` on `abap_debug_session`).
2. Open a **new** SAP GUI session via `SM51 → select application server → Goto → Remote Logon → Normal`.
3. Run the program in that new window. Sessions opened from SAP Logon / SAP Business Client, or
   opened before the capture started, do not stop at the external breakpoint.
4. For a selection-screen report, the breakpoint fires after Execute (at `START-OF-SELECTION`),
   not while the selection screen is displayed.

If the capture times out although the program finished: the run happened in the wrong session,
with the wrong user, or outside the window. Re-check with the user, then re-arm — a timed-out
capture no longer listens.

## Rule 5 — Drive and clean up

- `abap_debug_session(action="step", stepMode:"over"|"into"|"out", count, until)` — a long loop
  returns a compact `trace` of `path`/`changes` past 20 steps; a step with `frame: entered` /
  `returned` / `changed` crossed a method boundary.
- `action="inspect"` — read variables at the current position; name specific ones when a frame
  has more than 60 variables (only the first 60 are returned by default).
- `action="continue"` with `toLine` — run to a line in the current source; useful to skip a loop
  and count the filled table at its end.
- `action="end"` — release. It can fail with `stepContinue failed / terminateDebuggee failed:
  ADT network error: ... timeout`; then `release.released = false` and the program stays suspended
  until SAP's debugger timeout ends it. Tell the user to press **Leave Debugger** in SAP GUI.
- One interactive session per SAP user per server; do not start a second one while one is held.

## Rule 6 — Return to neutral after every run

After `abap_debug_session(action="end")` — **especially** when `release.released=false` — never
declare "debug cleaned" from the `end` result alone. Verify with `abap_breakpoints(action="list")`.
A clean state is: no `yours` entry, `leftovers: []`, `listener: "none"`. This is the gate before
any new `abap_debug` arm.

Cleanup ladder, in order:

1. `released: true` **and** the list is clean → done.
2. `yours` still listed → call `abap_debug_session(action="end")` again. `abap_breakpoints(action="delete")` is also safe here: the delete handler keeps a listener that an active session of this server still needs.
3. `leftovers` present → `abap_breakpoints(action="delete")` removes external breakpoints left by a crashed capture; re-list.
4. `listener: "another debugger: ..."` → **do not auto-delete**. This is a suspended SAP GUI debuggee or another IDE holding the user. Deleting the listener by user also ends external debugging for Eclipse/VS Code and drops the user's external breakpoints, and it does **not** release a suspended program. Tell the user to press **Leave Debugger** in SAP GUI (or close/log off that SAP session), then re-list. SAP's own debugger timeout eventually releases it too.
5. Re-list once more; report the verified `listener`/`leftovers` before starting a new capture.

Never kill the ADAM MCP process to clear state — a killed capture leaves leftovers; use
`abap_breakpoints(action="delete")` instead.

## Rule 7 — Read the failure text, do not guess

| Observed text | Cause and response |
|---|---|
| `Another debugger is already listening ... global debugging scope` | Suspended SAP GUI debuggee or another IDE. Rule 1: user must Leave Debugger / log off; listener delete alone is insufficient. |
| `reason: "conflict: ..."` right after arming, SAP GUI shows **ED702 "Debugging hindered by external development environment"** | External breakpoints set in the ABAP Editor exclude ADT's for the same user. In SAP GUI: `Utilities → Breakpoints → Delete` (`Ctrl+Shift+F12`), then retry. |
| `noSessionAttached` from a different debugger surface | Internal debugger (`/h`), not ADAM's external one. Never type `/h`. |
| `trigger:"none"` times out although the program ran | Wrong session type (GUI instead of SM51 RFC), wrong user, or run outside `timeoutSeconds`. Re-check the run contract; re-arm. |
| `trigger:"none"` stops at the wrong request | Capture takes the first request of that user that hits a breakpoint. Start it after the page loads; breakpoint only in code the intended action runs. |
| `Cannot tell which SAP user this session runs as` | SAP_BASIS 7.50 and older lacks `systeminformation`: set `SAP_USER` to the logon user. |
| `SAP could not reach the debuggee's application server` / `did not route the debugger session` | Missing `S_RFC` for `SADT_REST` (Eclipse ADT needs it too), or release cannot forward ADT between servers (7.50). Nothing attached; the program waits until SAP gives up. |
| `SAP attached in non-exclusive mode` | All exclusive-debug work processes taken (`rdisp/wpdbug_max_no`); ADAM releases the program untouched because each step would commit open DB changes. Retry later or have Basis raise the limit. |
| `ADT network error: ... fetch failed / unable to get local issuer certificate` | Corporate proxy/TLS: set `SAP_PROXY` and `NODE_EXTRA_CA_CERTS` in the MCP server's `env` block (Node reads it at startup, not from `--env-file`). Prefer the CA over `SAP_INSECURE=true` (test systems only). |
| `running: true` | The call exceeded `ABAP_DEBUGGER_CALL_SECONDS` (default 50) and continues in the background: `abap_debug_session(action:"wait")`; `end` stops it. |
| Client says `timed out after 60s` | The client cancelled the call, which stops the capture. Set `ABAP_DEBUGGER_CALL_SECONDS` a few seconds below the client's timeout. |
| Held session disappeared | Released after `ABAP_DEBUGGER_IDLE_SECONDS` (default 120), `ABAP_DEBUGGER_SESSION_SECONDS` (default 900), server stop/redeploy, or SAP's `rdisp/max_debug_lazy_time` (default 600). Start a new one. |
| `Debug captures running on this server: 4 of 4` | Caps `ABAP_DEBUGGER_MAX_CAPTURES` / `MAX_SESSIONS` (default 4 each); one active capture per SAP user. Wait or raise within SAP's work-process budget. |
| Message breakpoint never stops | It catches `MESSAGE` statements only — not RAP `new_message` / BAPI return tables. Find the number with `abap_source(grep)` instead. |
| Unit tests finished without reaching the breakpoint | No harmless test covers the code, or activation has not reached the application server yet (wait ~1 minute); use `triggerName` or `trigger:"none"`. |
| `trigger:"http"` 403 "blocked by UCON" | Service published moments ago on BTP ABAP; verify the URL on its own and retry. |

## Rule 8 — Config facts that change the setup

Check the ADAM server's `env` block in `mcp.json` before blaming the tool:

- `SAP_URL`, `SAP_CLIENT`, `SAP_USER`, `SAP_PASSWORD` — on-prem Basic. SSO via `SAP_COOKIE_FILE` /
  `SAP_COOKIE_STRING` (on 7.50 and older also set `SAP_USER`); BTP ABAP via
  `SAP_BTP_SERVICE_KEY_FILE` / `SAP_BTP_SERVICE_KEY`.
- `SAP_PROXY` (not `HTTPS_PROXY` — ADAM ignores it) and `NODE_EXTRA_CA_CERTS` for inspecting
  proxies. `SAP_INSECURE=true` only on test systems.
- `ABAP_DEBUGGER_SAP_GUI` — `true` gives the SM51 steps for `trigger:"none"`; default false behind
  a BTP service key / Internet destination, else true. On systems without SAP GUI (BTP ABAP,
  S/4HANA Cloud Public) do **not** instruct SM51 — prefer `unittest` / `http` triggers.
- Time budgets: `ABAP_DEBUGGER_CALL_SECONDS` (50), `IDLE_SECONDS` (120), `SESSION_SECONDS` (900),
  `MAX_SECONDS` (600 ceiling), `MAX_RESULT_KB` (200 ceiling), and the tool's `timeoutSeconds`
  (default 60; 300 with `trigger:"none"`, max 900). Keep `IDLE_SECONDS` below SAP's
  `rdisp/max_debug_lazy_time` so ADAM releases before SAP aborts the request.
- `ABAP_DEBUGGER_MAX_CAPTURES` / `MAX_SESSIONS` (4 each), `ABAP_DEBUGGER_STATE_DIR` (where a
  killed capture records leftovers), `ABAP_DEBUGGER_SAP_GUI`.

## Rule 9 — Avoid brute-force stepping

A long loop can exceed the time budget before reaching the line you want (a step costs 2-3 SAP
requests; one call answers within `CALL_SECONDS`). Prefer:

- `abap_debug` breakpoint `condition` (e.g. `sy-tabix = 500`) or `hits` (1-50) to reach one
  iteration instead of all of them;
- `watch` ({ variable, condition?, hits? }) to stop where a variable changes — it is set at the
  first stop, so the variable must be visible there;
- `abap_debug_session(action:"continue", toLine:<N>)` to skip a loop, then `inspect` the filled
  table at its end; `until: {line}` / `{changed}` to bound a step run.

Display limits: stops show structures one level deep and tables with a line count plus the first
5 rows; `inspect` with named variables reads any of them in full. A value equal to the previous
step's reads `unchanged`; `frame: entered|returned|changed` means a method boundary was crossed.

## Outcome report

State the observed facts: whether the breakpoint hit, procedure/line/server, variables read,
steps/trace, release status, and the verified post-run `abap_breakpoints list` state. Report a
`released: false` release as a still-suspended program with the required SAP GUI action. Do not
claim a fix works until the program is released and the source is activated/verified through the
normal SAP workflow.
