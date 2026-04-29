---
name: d6n
description: Install and configure the D6N MCP tools for buying and selling your session data. Use when the user wants to connect their AI agent to the D6N network for buying and selling their AI agent sessions.
argument-hint: [replace-keys|reauthorize]
allowed-tools: Bash, AskUserQuestion
---

<!--
Copyright 2026 Neosphere Inc
SPDX-License-Identifier: Apache-2.0
-->

# D6N MCP Tools

Connect your AI agent to the [D6N](https://d6n.ai) network for buying and selling AI agent sessions and other data. D6N exposes MCP tools for OAuth authorization, session discovery, purchase via MPP, and session management.

## Workflow

If `$ARGUMENTS` is `replace-keys` or `reauthorize`, skip to **Reauthorize**.

Do not ask the user to visit D6N and manually generate Buyer/Seller keys. Use the OAuth tools below.

### Step 1: Choose install scope and D6N role

Ask the user:

> Would you like to install D6N for **this project only** (local) or **all projects** (global)?
>
> Do you want this AI agent to **buy** data on D6N, **sell** data on D6N, or do **both**?

- **Local** (default): config is stored in the project's `.claude.json`.
- **Global**: config is stored in user-level settings.

Derive OAuth scopes from the user's answer. Do not ask them for raw OAuth scope strings.

| User intent | OAuth `scopes` | Suggested `client_id` suffix |
|-------------|----------------|------------------------------|
| Buy data | `buy` | `buy` |
| Sell data | `sell` | `sell` |
| Buy and sell data | `buy sell` | `buy+sell` |

Pick a stable `client_id` shown on the consent page, such as `Claude Code (<project-name>) - buy+sell`. Include the selected role in the `client_id` so the approval page is clear. Reuse the exact same `client_id` and derived `scopes` for `create_oauth` and `check_oauth`.

### Step 2: Add a public MCP connection

Use `https://d6n.ai/mcp` unless an existing D6N MCP config already uses another URL.

Local:

```bash
claude mcp add --transport http d6n https://d6n.ai/mcp
```

Global:

```bash
claude mcp add --transport http d6n https://d6n.ai/mcp -s user
```

If `d6n` is already configured, preserve its scope and URL and continue to OAuth. If the client does not expose newly added MCP tools until reload, tell the user to reload/restart the AI client and run the D6N setup again. Do not fall back to manual key generation.

### Step 3: Create OAuth request

Call the D6N MCP tool:

```text
create_oauth(client_id="<CLIENT_ID>", scopes="<SCOPES>")
```

It returns:

```json
{"oauth_id": "...", "oauth_url": "https://d6n.ai/oauth?oauth_id=..."}
```

Send the `oauth_url` to the user and ask them to open it, sign in, and approve. Tell them the consent page will show that they can go back to their AI agent after approval, then redirect to their D6N profile.

### Step 4: Poll for approval

Poll the D6N MCP tool every few seconds:

```text
check_oauth(client_id="<SAME_CLIENT_ID>", scopes="<SAME_SCOPES>", oauth_url="<OAUTH_URL>")
```

- Pass the exact same `client_id` and `scopes` used for `create_oauth`.
- `{"status": "pending"}` means keep waiting.
- Rejected or expired requests return 404; create a fresh OAuth request.
- Approved responses include `auth_key` / `obo_key` with a `cli_ke...` credential.
- The approved result is one-shot. Once you receive the credential, immediately install it and do not call `check_oauth` again for the same request.

### Step 5: Re-add MCP with the OAuth credential

Let `<CLI_KEY>` be the returned `auth_key` or `obo_key`.

Local:

```bash
claude mcp remove d6n 2>/dev/null || true
claude mcp add --transport http d6n https://d6n.ai/mcp \
  -H "Authorization: Bearer <CLI_KEY>"
```

Global:

```bash
claude mcp remove d6n -s user 2>/dev/null || true
claude mcp add --transport http d6n https://d6n.ai/mcp -s user \
  -H "Authorization: Bearer <CLI_KEY>"
```

Do not print the full credential in the final answer.

### Step 6: Verify

```bash
claude mcp list 2>/dev/null | grep d6n
```

Expected output:

```text
d6n: https://d6n.ai/mcp (HTTP) - Connected
```

If connected, tell the user D6N is installed and summarize what is available.

## Available Tools

Each tool is a separate MCP method. After OAuth setup, authenticated tools use the `Authorization: Bearer <cli_ke...>` OBO credential. Session IDs are passed as the `datum_id` param in MCP calls.

### OAuth setup

| Tool | Description | Required params | Optional params | Auth |
|------|-------------|-----------------|-----------------|------|
| `create_oauth` | Create a short-lived user consent URL. | `client_id`, `scopes` | - | none |
| `check_oauth` | Check consent status and return a one-shot CLI-prefixed OBO credential when approved. | `client_id`, `scopes` | `oauth_url` or `oauth_id` | none |

### Discovery & purchase

| Tool | Description | Required params | Optional params | Auth |
|------|-------------|-----------------|-----------------|------|
| `search_sessions` | Search the D6N marketplace within your seller scope. | - | - | `sell` grant |
| `buy_session` | Purchase a session via MPP. First call returns a 402 with price and accepted methods; retry with the SPT credential in `Payment-Credential` to complete. Returns the session directly if already owned. | `datum_id` | - | `buy` grant |

### Session management

| Tool | Description | Required params | Optional params | Auth |
|------|-------------|-----------------|-----------------|------|
| `create_session` | Upload a new session. | `file_base64`, `filename`, `category`, `subcategory`, `file_format` | `description`, `modality`, `tags`, `languages`, `price_cents` (default `0` = free), `source`, `open_to_public` (default `false`) | `sell` grant |
| `get_session` | Fetch one session by ID. Accessible to owner or any buyer. | `datum_id` | - | `buy` or `sell` grant |
| `list_sessions` | List your sessions. | - | `owned` (default `true`; set `false` to list purchased sessions), `limit` (1-50, default 50) | `buy` or `sell` grant |
| `update_session` | Update metadata on a session you own. Only provided fields change. | `datum_id` | `category`, `subcategory`, `file_format`, `description`, `modality`, `tags`, `languages`, `price_cents`, `source`, `open_to_public` | `sell` grant |
| `delete_session` | Permanently delete a session you own, including its file. | `datum_id` | - | `sell` grant |

## Reauthorize

Use this flow when `$ARGUMENTS` is `replace-keys` or `reauthorize`.

1. Read the current config:

```bash
claude mcp get d6n 2>/dev/null
```

If `d6n` is not configured, run the normal workflow.

2. Preserve the current install scope and MCP URL.
3. Run the OAuth workflow from **Step 1**, asking whether the user wants this agent to buy, sell, or do both on D6N.
4. Remove and re-add `d6n` with the new `Authorization: Bearer <CLI_KEY>` header.
5. Verify with `claude mcp list`.

Do not ask the user to paste Buyer/Seller keys during reauthorization.

## Uninstall

Local:

```bash
claude mcp remove d6n
```

Global:

```bash
claude mcp remove d6n -s user
```
