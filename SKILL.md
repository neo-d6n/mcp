---
name: d6n
description: Optional shortcut to install or reauthorize the D6N MCP server for the active AI coding agent using a human-created D6N agent auth code.
argument-hint: [reauthorize|replace-keys]
allowed-tools: Bash, AskUserQuestion
---

<!--
Copyright 2026 Neosphere Inc
SPDX-License-Identifier: Apache-2.0
-->

# D6N MCP Install Shortcut

This skill is optional. D6N does not require skill installation: agents can
discover `https://d6n.ai/.well-known/agent.yml` or `https://d6n.ai/llms.txt`,
claim a human-created agent auth code, and configure `https://d6n.ai/mcp` directly. Use this skill only as
a convenience installer or reauthorization helper for clients that support
skills. After install, the D6N MCP server itself exposes the working tools.

D6N grants access through a human-created six-digit agent auth code. The
claimed result is a `cli_ke...` OBO credential scoped to `buy`, `sell`, or
both. The six-digit code is not the final credential and is safe to ask for;
the returned `auth_key` is the credential and must stay secret. Do not ask the
user to paste credentials.

Do not use any old OAuth URL, callback URL, or approval polling flow. The only
agent authorization flow is: owner creates a six-digit code at
`/aiauth/create`, agent claims it once at `/aiauth/claim/{code}`, then the
agent uses the returned scoped Bearer credential.

## What To Install

Detect the active client before doing anything:

1. If any `CODEX_*` environment variable exists, configure **Codex**.
2. Else if any `CLAUDE_*` or `CLAUDE_CODE_*` environment variable exists, configure **Claude Code**.
3. Else if only `codex` or only `claude` exists on `PATH`, configure that client.
4. Else ask which client to configure.

Use the matching MCP endpoint:

- Production: `https://d6n.ai/mcp`
- Local dev: `http://127.0.0.1:8991/mcp`

Use the matching agent auth HTTP origin:

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

Do not ask for raw scope strings if the user already answered in normal language.

Build a stable `client_id`:

- Codex: `Codex <project-name> <scope-label>`
- Claude Code: `Claude Code <project-name> <scope-label>`

D6N sanitizes this value server-side. Keep it short and readable.

### 2. Ask The Human For A D6N Agent Auth Code

Ask the human to open:

```text
https://d6n.ai/aiauth/create
```

or, for local dev:

```text
http://127.0.0.1:8990/aiauth/create
```

Tell them to log in, enter the suggested agent name (`<CLIENT_ID>`), select the requested `buy` and/or `sell` scope, click Create, and return with the six-digit code. If they already have a code, use it.

The owner may click Cancel before creating the code. A created code is valid
for 1 hour and can be claimed once. A successful claim returns a credential
that currently expires after 72 hours. Reauthorizing with the same owner and
agent name replaces older active D6N OBO credentials for that agent name.

### 3. Claim The Code

After the user gives the six-digit code, call:

```bash
curl -sS "$D6N_HTTP_ORIGIN/aiauth/claim/<CODE>"
```

Results:

- `404`: the code expired, was mistyped, or was already consumed. Ask the human to create a new code.
- Approved result: contains `auth_key`, `client_id`, `scopes`, and `expiration_time`. Use `auth_key` immediately. The result is one-shot and the credential expires after 72 hours.

Never print the full credential.

### 4. Configure MCP

Let `<CLI_KEY>` be the approved `cli_ke...` credential. If the host cannot
store secrets safely, keep the key only in the current execution context and
tell the user they will need to reauthorize when the context ends or the
72-hour credential expires.

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

Then ask the agent to call a read-only D6N MCP tool. For a `sell` or
`buy sell` credential, prefer:

```text
list_listings(owned=true, limit=1)
```

For a `buy` credential, prefer:

```text
list_listings(owned=false, limit=1)
```

Expected: the call completes without an auth/configuration error. An empty list
is valid.

## Current MCP Tools

After the server is configured, use the MCP tools exposed by the live D6N
server. The current production surface covers listing search/create/manage and
seller order fulfillment.

Listing creation tools require `sell` scope:

- `create_data_listing(files, title, description)`
- `create_physical_good_listing(files, title, description, price_cents, condition)`
- `create_saas_listing(files, title, description, service_url, price_cents, billing_interval)`
- `create_vacation_rental_listing(files, title, price_cents, location_address, location_city, location_country, max_guests, bedrooms, bathrooms, availability_start, availability_end)`
- `create_appointments_listing(files, title, description, price_cents, service_duration_minutes, meeting_platform)`

Listing read/manage tools:

- `search_listings(q, listing_type, tags_any, languages_any, amenities_any, price_cents_min, price_cents_max, currency, category, location_city, location_region, location_country, service_type, sort, mode, limit, cursor)`
- `list_listings(owned=true, limit=50)`: list listings created by the authenticated user.
- `list_listings(owned=false, limit=50)`: list listings purchased by the authenticated user.
- `get_listing(datum_id)`: retrieve a listing visible to the authenticated user.
- `delete_listing(datum_id)`: permanently delete a listing owned by the authenticated user; requires `sell` scope and ownership.

Seller order tools:

- `get_order(order_id)`
- `list_orders(role="seller", limit=20)`
- `set_order_shipping_label(order_id, shipping_label_id)`
- `set_order_tracking(order_id, tracking_number)`
- `mark_order_delivered(order_id)`

D6N does not currently expose an MCP `buy_listing` tool. Buyer purchase flows
use `POST https://d6n.ai/buy` with a `buy` credential.

Use `datum_id` for listing IDs because the backend API still names the resource
that way. For create tools, `files` is required for every listing type and
should contain one or more objects like
`{"filename":"example.pdf","file_base64":"..."}`. Do not use the old `session`
or open-order tool names unless the server explicitly exposes them in the
active MCP tool list.

## Reauthorize

For `reauthorize` / `replace-keys`:

1. Detect the active client.
2. Check whether `d6n` is already configured with `codex mcp get d6n` or `claude mcp get d6n`.
3. Run the agent auth code flow again with the requested scopes.
4. Replace only the active client's `d6n` MCP entry after the new code is
   claimed successfully.
5. Tell the user to restart the active client and call a read-only listing tool.

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
