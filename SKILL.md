---
name: d6n
description: Install or reauthorize the D6N MCP server for the active AI coding agent using the D6N OAuth consent flow.
argument-hint: [reauthorize|replace-keys]
allowed-tools: Bash, AskUserQuestion
---

<!--
Copyright 2026 Neosphere Inc
SPDX-License-Identifier: Apache-2.0
-->

# D6N MCP Install

Use this skill only to install or reauthorize the D6N MCP server. After install, the D6N MCP server itself exposes the working tools.

D6N grants access through a user-approved OAuth URL. The approved result is a `cli_ke...` OBO credential scoped to `buy`, `sell`, or both. Do not ask the user to create or paste credentials.

## What To Install

Detect the active client before doing anything:

1. If any `CODEX_*` environment variable exists, configure **Codex**.
2. Else if any `CLAUDE_*` or `CLAUDE_CODE_*` environment variable exists, configure **Claude Code**.
3. Else if only `codex` or only `claude` exists on `PATH`, configure that client.
4. Else ask which client to configure.

Use the matching MCP endpoint:

- Production: `https://d6n.ai/mcp`
- Local dev: `http://127.0.0.1:8991/mcp`

Use the matching OAuth HTTP origin:

- Production: `https://d6n.ai`
- Local dev: `http://127.0.0.1:8990`

Prefer production unless the current repo/session is clearly using the local D6N stack.

## Install Flow

If `$ARGUMENTS` is `reauthorize` or `replace-keys`, follow the same flow and replace the existing `d6n` MCP entry for the active client.

### 1. Ask For Scope

Ask the user whether this agent should:

- `buy`
- `sell`
- `buy sell`

Do not ask for raw OAuth scopes if the user already answered in normal language.

Build a stable `client_id`:

- Codex: `Codex <project-name> <scope-label>`
- Claude Code: `Claude Code <project-name> <scope-label>`

D6N sanitizes this value server-side. Keep it short and readable.

### 2. Create OAuth URL

```bash
curl -sS -X POST "$D6N_HTTP_ORIGIN/oauth/create" \
  -H 'Content-Type: application/json' \
  -d '{"client_id":"<CLIENT_ID>","scopes":"<SCOPES>"}'
```

The response has:

```json
{"oauth_id":"...","oauth_url":"..."}
```

Send the `oauth_url` to the user. Ask them to open it, sign in, approve, and reply `done`.

### 3. Collect Approved Credential

After the user replies, call:

```bash
curl -sS "$D6N_HTTP_ORIGIN/oauth/check?client_id=<URL_ENCODED_CLIENT_ID>&scopes=<URL_ENCODED_SCOPES>&oauth_id=<OAUTH_ID>"
```

Results:

- `{"status":"pending"}`: ask the user to finish approval, then check again.
- `404`: OAuth request expired, was rejected, or was already consumed. Start over.
- Approved result: contains `auth_key` or `obo_key`. Use that value immediately. The result is one-shot.

Never print the full credential.

### 4. Configure MCP

Let `<CLI_KEY>` be the approved `cli_ke...` credential.

#### Codex

Codex stores MCP config in `~/.codex/config.toml` and supports remote HTTP bearer auth through an environment variable.

```bash
CLI_KEY='<CLI_KEY>'
mkdir -p "$HOME/.config/d6n"
chmod 700 "$HOME/.config/d6n"
cat > "$HOME/.config/d6n/mcp.env" <<EOF
export D6N_MCP_BEARER='$CLI_KEY'
EOF
chmod 600 "$HOME/.config/d6n/mcp.env"

PROFILE="$HOME/.zshrc"
if [ -n "${BASH_VERSION:-}" ]; then PROFILE="$HOME/.bashrc"; fi
grep -q 'D6N MCP bearer for Codex' "$PROFILE" 2>/dev/null || cat >> "$PROFILE" <<'EOF'

# D6N MCP bearer for Codex
source "$HOME/.config/d6n/mcp.env"
EOF

. "$HOME/.config/d6n/mcp.env"
codex mcp remove d6n 2>/dev/null || true
codex mcp add d6n --url "$D6N_MCP_URL" --bearer-token-env-var D6N_MCP_BEARER
```

#### Claude Code

Ask whether to install for this project or all projects.

This project:

```bash
claude mcp remove d6n 2>/dev/null || true
claude mcp add --transport http --scope local \
  --header "Authorization: Bearer <CLI_KEY>" \
  d6n "$D6N_MCP_URL"
```

All projects:

```bash
claude mcp remove d6n --scope user 2>/dev/null || true
claude mcp add --transport http --scope user \
  --header "Authorization: Bearer <CLI_KEY>" \
  d6n "$D6N_MCP_URL"
```

Use Claude project scope only if explicitly requested, because `.mcp.json` is intended to be shareable and must not contain secrets.

### 5. Verify

The user must restart the active AI client before new MCP tools appear.

After restart:

```bash
# Codex
. "$HOME/.config/d6n/mcp.env" 2>/dev/null || true
codex mcp list
codex mcp get d6n

# Claude Code
claude mcp list
claude mcp get d6n
```

Then ask the agent to call the D6N `whoami` MCP tool. Expected sentence:

```text
You are <client_id> that can <scope> for user <username>.
```

## Current MCP Tools

Identity:

- `whoami`: confirms the configured D6N bearer token, client id, scope, and username.

Open orders:

- `create_buy_order(amount_cents, description)`: create an open buy order. Requires `buy` scope.
- `create_sell_order(amount_cents, description)`: create an open sell order. Requires `sell` scope.
- `delete_order(order_id)`: close an order the user/agent is allowed to manage.

Marketplace sessions:

- `search_sessions`: search available sessions.
- `buy_session(datum_id)`: buy a session through MPP/payment challenge flow.
- `create_session(...)`: upload a sellable session. Requires `sell` scope.
- `update_session(datum_id, ...)`: update a session you own. Requires `sell` scope.
- `delete_session(datum_id)`: delete a session you own. Requires `sell` scope.
- `get_session(datum_id)`: retrieve a session visible to the authenticated user.
- `list_sessions(owned=true, limit=50)`: list owned or purchased sessions.

Use `datum_id` for session IDs because the backend API still names the resource that way.

## Reauthorize

For `reauthorize` / `replace-keys`:

1. Detect the active client.
2. Check whether `d6n` is already configured with `codex mcp get d6n` or `claude mcp get d6n`.
3. Run the OAuth flow again with the requested scopes.
4. Replace only the active client's `d6n` MCP entry.
5. Tell the user to restart the active client and call `whoami`.

## Uninstall

Codex:

```bash
codex mcp remove d6n
```

Claude Code local/current project:

```bash
claude mcp remove d6n
```

Claude Code user/global:

```bash
claude mcp remove d6n --scope user
```
