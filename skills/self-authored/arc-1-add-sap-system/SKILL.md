---
name: arc-1-add-sap-system
description: Add a new SAP system logon to the ARC-1 MCP configuration (VS Code User mcp.json + Copilot CLI mcp-config.json) as a new named server entry. Use when asked to "add SAP system", "add another SAP logon", "add client 120", "add trial system", "new SAP connection", or "register SAP system for ARC-1".
---

# Add SAP System to ARC-1

Add a new SAP system as a named ARC-1 MCP server entry in both config files.
One server entry = one SAP logon (URL + user + client). Never edit the
existing `arc-1` entry in place — always add a sibling entry.

## Files

| File | Schema | Server key style |
|---|---|---|
| `C:\Users\narue\AppData\Roaming\Code\User\mcp.json` | `"servers"` | `"arc-1-<suffix>"` with `"type": "stdio"`, `"command": "npx"` |
| `C:\Users\narue\.copilot\mcp-config.json` | `"mcpServers"` | same suffix, `"command": "C:\\Program Files\\nodejs\\npx.cmd"`, plus `"tools": ["*"]` |

## Smart Defaults (apply silently, do NOT ask)

| Setting | Default | Rationale |
|---|---|---|
| Entry suffix | lowercase SID + client, e.g. `gsd120`, `trial` | Matches ADT destination naming |
| `SAP_URL` | `https://vhgchgwdwd01.sap.pttgcgroup.com:44381` for GSD clients | Same ICM host serves clients 110 and 120 |
| `SAP_USER` | `Z0012353` | Same user on all known systems |
| `SAP_LANGUAGE` | `EN` | User's logon language |
| `SAP_INSECURE` | `false` | TLS verified via CA bundle |
| `NODE_EXTRA_CA_CERTS` | `C:\Users\narue\Documents\SAP\Cert\sap-pttgcgroup-ca-bundle.crt` | Bundle covers self-signed leaf + corporate root CA |
| `SAP_ALLOW_WRITES` | `true` | Matches existing policy |
| `SAP_ALLOWED_PACKAGES` | `$TMP,Z*,Y*` | Matches existing policy |
| `SAP_ALLOW_TRANSPORT_WRITES` | `true` | Matches existing policy |
| `SAP_ALLOW_DATA_PREVIEW` | `false` | Table content stays off |
| `SAP_ALLOW_FREE_SQL` | `false` | Freestyle SQL stays off |
| `SAP_ALLOW_GIT_WRITES` | `false` | Git mutations stay off |
| `cwd` (VS Code entry only) | `C:\Users\narue\.copilot\plugin-data\_direct\9eba23c62bd8ea4c55b38e181310d1a2a81deb9f741a53e631cefc134940fd95` | Working dir whose `.env` ARC-1 loads via dotenv |
| Password (VS Code entry) | `"${input:sap-password}"` | Reuses the shared `sap-password` prompt input; do NOT create a second password input |
| Password (CLI entry) | `"${SAP_PASSWORD}"` | Resolved from the `SAP_PASSWORD` User environment variable |

## Input (ask only what is missing)

1. **SAP system** — SID/host and client (e.g. "GSD client 120", "trial `https://....abap.us10.hana.ondemand.com`"). If the host is a new one, ask for the full HTTPS ADT URL.
2. **Entry suffix** — propose `<sid><client>` (e.g. `gsd120`); accept the user's name if given.
3. **User** — only if different from `Z0012353`.

Do NOT ask about safety flags, CA bundle, language, or `cwd` — use Smart Defaults.

## Steps

### 1. Validate the new entry name

Read both config files. The suffix must not collide with an existing server key
in either file. If it collides, propose `<suffix>-2` or ask for another name.

### 2. Add the VS Code entry

In `C:\Users\narue\AppData\Roaming\Code\User\mcp.json` under `"servers"`,
add (replace `<suffix>`, `<url>`, `<user>`, `<client>`):

```json
"arc-1-<suffix>": {
  "type": "stdio",
  "command": "npx",
  "args": ["-y", "arc-1@latest"],
  "cwd": "C:\\Users\\narue\\.copilot\\plugin-data\\_direct\\9eba23c62bd8ea4c55b38e181310d1a2a81deb9f741a53e631cefc134940fd95",
  "env": {
    "SAP_URL": "<url>",
    "SAP_USER": "<user>",
    "SAP_PASSWORD": "${input:sap-password}",
    "SAP_CLIENT": "<client>",
    "SAP_LANGUAGE": "EN",
    "SAP_INSECURE": "false",
    "NODE_EXTRA_CA_CERTS": "C:\\Users\\narue\\Documents\\SAP\\Cert\\sap-pttgcgroup-ca-bundle.crt",
    "SAP_ALLOW_WRITES": "true",
    "SAP_ALLOWED_PACKAGES": "$TMP,Z*,Y*",
    "SAP_ALLOW_TRANSPORT_WRITES": "true",
    "SAP_ALLOW_DATA_PREVIEW": "false",
    "SAP_ALLOW_FREE_SQL": "false",
    "SAP_ALLOW_GIT_WRITES": "false"
  }
}
```

Do NOT add a new `inputs` entry — reuse the existing `sap-password` input.

### 3. Add the Copilot CLI entry

In `C:\Users\narue\.copilot\mcp-config.json` under `"mcpServers"`, add:

```json
"arc-1-<suffix>": {
  "type": "stdio",
  "command": "C:\\Program Files\\nodejs\\npx.cmd",
  "args": ["-y", "arc-1@latest", "serve"],
  "tools": ["*"],
  "env": {
    "SAP_URL": "<url>",
    "SAP_USER": "<user>",
    "SAP_PASSWORD": "${SAP_PASSWORD}",
    "SAP_CLIENT": "<client>",
    "SAP_LANGUAGE": "EN",
    "SAP_INSECURE": "false",
    "NODE_EXTRA_CA_CERTS": "C:\\Users\\narue\\Documents\\SAP\\Cert\\sap-pttgcgroup-ca-bundle.crt",
    "SAP_ALLOW_WRITES": "true",
    "SAP_ALLOWED_PACKAGES": "$TMP,Z*,Y*",
    "SAP_ALLOW_TRANSPORT_WRITES": "true",
    "SAP_ALLOW_DATA_PREVIEW": "false",
    "SAP_ALLOW_FREE_SQL": "false",
    "SAP_ALLOW_GIT_WRITES": "false"
  }
}
```

### 4. Validate

Parse both files as JSON and confirm the new key exists in each.
Run `copilot mcp list` and confirm the new server appears.

### 5. Tell the user to activate it

- VS Code: `Ctrl+Shift+P` → `Developer: Reload Window`, then `MCP: List Servers` → enable `arc-1-<suffix>` (disable the others if only one system should be active).
- CLI: `copilot mcp get arc-1-<suffix>` to verify.

## Rules

- Never store a real password in either file. VS Code uses `${input:sap-password}` (secret storage); CLI uses `${SAP_PASSWORD}` (User env var).
- Never change the existing `arc-1` entry when adding a new system.
- Never touch `C:\Users\narue\.copilot\installed-plugins\arc-1\arc-1\mcp.json` (must stay `"mcpServers": {}` to avoid duplicate servers in VS Code).
- For BTP trial hosts (`*.hana.ondemand.com`), the same template applies — only `SAP_URL` changes. If TLS fails on a new host, report it; do NOT silently set `SAP_INSECURE=true`.
