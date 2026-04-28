---
name: d6n
description: Install and configure the D6N MCP tools for buying and selling your session data. Use when the user wants to connect their AI agent to the D6N network for buying and selling their AI agent sessions.
argument-hint: [replace-keys]
allowed-tools: Bash, AskUserQuestion
---

<!--
Copyright 2026 Neosphere Inc
SPDX-License-Identifier: Apache-2.0
-->

# D6N MCP Tools

Connect your AI agent to the [D6N](https://d6n.ai) network for buying and selling AI agent sessions (and other data). D6N exposes a set of tools over the Model Context Protocol, covering session discovery, purchase (via MPP), and session management.

## Workflow

If `$ARGUMENTS` is `replace-keys`, skip to the **Replace Keys** section below.

### Step 1: Ask for keys

Tell the user they need two keys from d6n.ai, then ask them to paste the keys in the chat:

> To connect to D6N, you need two keys from your d6n.ai account.
>
> 1. Go to [d6n.ai](https://d6n.ai) and sign in
> 2. Click your avatar (top-right circle) to open your profile
> 3. Select **Keys** from the left sidebar
> 4. Copy your **Seller Key** and **Buyer Key**
>
> Please paste both keys here (you can paste them together in one message).

Wait for the user to provide at least one key. The user may paste both keys in a single message. Identify them by prefix:

- **Buyer Key** starts with `buy_ke`
- **Seller Key** starts with `sell_ke`

If the user only provides one key, proceed with just that key. Only include headers for keys the user has provided.

### Step 2: Ask scope

Ask the user:

> Would you like to install D6N for **this project only** (local) or **all projects** (global)?

- **Local** (default): config is stored in the project's `.claude.json`, private to this project.
- **Global**: config is stored in your user-level settings, available across all projects.

### Step 3: Install

Run the following command with the user's keys, adding `-s user` if they chose global:

**Local (default):**

```bash
claude mcp add --transport http d6n https://d6n.ai/mcp \
  -H "X-Seller-Key: <SELLER_KEY>" \
  -H "X-Buyer-Key: <BUYER_KEY>"
```

**Global:**

```bash
claude mcp add --transport http d6n https://d6n.ai/mcp -s user \
  -H "X-Seller-Key: <SELLER_KEY>" \
  -H "X-Buyer-Key: <BUYER_KEY>"
```

All provided keys are sent with every tool call.

### Step 4: Verify

```bash
claude mcp list 2>/dev/null | grep d6n
```

Expected output:

```
d6n: https://d6n.ai/mcp (HTTP) - Connected
```

If it shows `Connected`, tell the user the install succeeded and summarize what's available (see Available Tools below).

### Step 5: Inform user about key rotation

After successful install, tell the user where their config lives and how to cycle keys. Tailor the message to the scope they chose:

**If local:**

> Your D6N MCP config is stored in this project's `.claude.json`. If you ever need to cycle your keys, run:
>
> ```
> claude mcp remove d6n
> claude mcp add --transport http d6n https://d6n.ai/mcp \
>   -H "X-Seller-Key: <NEW_SELLER_KEY>" \
>   -H "X-Buyer-Key: <NEW_BUYER_KEY>"
> ```

**If global:**

> Your D6N MCP config is stored in your user-level settings (`~/.claude.json`). If you ever need to cycle your keys, run:
>
> ```
> claude mcp remove d6n -s user
> claude mcp add --transport http d6n https://d6n.ai/mcp -s user \
>   -H "X-Seller-Key: <NEW_SELLER_KEY>" \
>   -H "X-Buyer-Key: <NEW_BUYER_KEY>"
> ```

## Available Tools

Each tool is a separate MCP method (no umbrella dispatchers). Headers required per tool:

- **`X-Seller-Key`** — search and session-management ops (you as a seller/owner).
- **`X-Buyer-Key`** — buy ops (you as a buyer).
- Some read tools accept either.

Session IDs are passed as the `datum_id` param in MCP tool calls.

### Discovery & purchase

| Tool | Description | Required params | Optional params | Auth |
|------|-------------|-----------------|-----------------|------|
| `search_sessions` | Search the D6N marketplace within your seller scope. | — | — | seller |
| `buy_session` | Purchase a session via MPP. First call returns a 402 with price + accepted methods; retry with the SPT credential in `Payment-Credential` to complete. Returns the session directly if already owned. | `datum_id` | — | buyer |

### Session management (your own sessions)

| Tool | Description | Required params | Optional params | Auth |
|------|-------------|-----------------|-----------------|------|
| `create_session` | Upload a new session. | `file_base64`, `filename`, `category`, `subcategory`, `file_format` | `description`, `modality`, `tags`, `languages`, `price_cents` (default `0` = free), `source`, `open_to_public` (default `false`) | seller |
| `get_session` | Fetch one session by ID. Accessible to owner or any buyer. | `datum_id` | — | buyer or seller |
| `list_sessions` | List your sessions. | — | `owned` (default `true`; set `false` to list sessions you've purchased), `limit` (1–50, default 50) | buyer or seller |
| `update_session` | Update metadata on a session you own. Only provided fields change. | `datum_id` | `category`, `subcategory`, `file_format`, `description`, `modality`, `tags`, `languages`, `price_cents`, `source`, `open_to_public` | seller |
| `delete_session` | Permanently delete a session you own (including its file). | `datum_id` | — | seller |



## Replace Keys

If `$ARGUMENTS` is `replace-keys`, run this flow instead of the install workflow above.

### Step 1: Read current config

```bash
claude mcp get d6n 2>/dev/null
```

If `d6n` is not configured, tell the user to run `/d6n` first to install.

Note the current scope (local or user) and URL from the output.

### Step 2: Ask for new keys

Tell the user:

> Paste the key(s) you want to replace. You can replace one or both.
>
> Provision new keys at [d6n.ai](https://d6n.ai) (avatar top-right > Keys).

- **Buyer Key** starts with `buy_ke`
- **Seller Key** starts with `sell_ke`

For any key the user does **not** provide, keep the existing value from the current config.

### Step 3: Re-add with updated keys

Remove and re-add, preserving the scope and URL from step 1:

```bash
claude mcp remove d6n 2>/dev/null || true
claude mcp add --transport http d6n <URL> \
  -H "X-Seller-Key: <SELLER_KEY>" \
  -H "X-Buyer-Key: <BUYER_KEY>"
```

Add `-s user` if the existing config was user-scoped.

### Step 4: Verify

```bash
claude mcp list 2>/dev/null | grep d6n
```

## Uninstall

```bash
claude mcp remove d6n
```
