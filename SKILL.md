---
name: d6n
description: Install and configure the D6N MCP tools for data procurement. Use when the user wants to connect their AI agent to the D6N network.
allowed-tools: Bash, AskUserQuestion
---

# D6N MCP Tools

Connect your AI agent to the [D6N](https://d6n.ai) data procurement network. D6N exposes three tools — `search`, `procure`, and `orders` — over the Model Context Protocol.

## Workflow

### Step 1: Ask for keys

Tell the user they need two keys from d6n.ai, then ask them to paste the keys in the chat:

> To connect to D6N, you need two keys from your d6n.ai account.
>
> 1. Go to [d6n.ai](https://d6n.ai) and sign in
> 2. Click your avatar (top-right circle) to open your profile
> 3. Select **API Keys** from the left sidebar
> 4. Copy your **Distribution Key** and **API Key**
>
> Please paste both keys here (you can paste them together in one message).

Wait for the user to provide both keys before proceeding. The user may paste both keys in a single message. Identify them by prefix:

- **API Key** starts with `api_ke`
- **Distribution Key** starts with `distr_ke`

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
  -H "Authorization: Bearer <DISTRIBUTION_KEY>" \
  -H "X-Api-Key: <API_KEY>"
```

**Global:**

```bash
claude mcp add --transport http d6n https://d6n.ai/mcp -s user \
  -H "Authorization: Bearer <DISTRIBUTION_KEY>" \
  -H "X-Api-Key: <API_KEY>"
```

Both headers are sent with every tool call.

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
>   -H "Authorization: Bearer <NEW_DISTRIBUTION_KEY>" \
>   -H "X-Api-Key: <NEW_API_KEY>"
> ```

**If global:**

> Your D6N MCP config is stored in your user-level settings (`~/.claude.json`). If you ever need to cycle your keys, run:
>
> ```
> claude mcp remove d6n -s user
> claude mcp add --transport http d6n https://d6n.ai/mcp -s user \
>   -H "Authorization: Bearer <NEW_DISTRIBUTION_KEY>" \
>   -H "X-Api-Key: <NEW_API_KEY>"
> ```

## Available Tools

### search

Search the D6N distributed network for available data resources.

### procure

Procure a data resource from the D6N network.

### orders

Manage data procurement orders.

**Actions:**

| Action | Description | Required params |
|--------|-------------|-----------------|
| `create` | Create a new order | `amount` (int). Optional: `creator_id`, `currency` (default "usd"), `description` |
| `read` | Get a single order | `order_id` (str) |
| `list` | List your orders | Optional: `limit` (int, default 20, max 50) |
| `delete` | Close an order | `order_id` (str) |

## Uninstall

```bash
claude mcp remove d6n
```
