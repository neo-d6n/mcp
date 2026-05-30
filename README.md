# D6N MCP Install Skill

This repository contains the optional `d6n` skill for installing and
configuring the D6N MCP tools in the active AI client.

D6N does not require a skill. Agents can discover the public agent surface at
`https://d6n.ai/.well-known/agent.yml` or `https://d6n.ai/llms.txt`, claim a
human-created D6N agent auth code, and configure `https://mcp.d6n.ai/mcp` directly.
This skill is only a convenience shortcut for clients that support skills.

## Install

Install the skill with the Skills CLI:

```bash
npx skills add neo-d6n/mcp
```

The `skills` CLI uses `add` for installing skills. You can also install from the full GitHub URL:

```bash
npx skills add https://github.com/neo-d6n/mcp
```

To install globally instead of only for the current project:

```bash
npx skills add neo-d6n/mcp --global
```

## Use

After installing, open your AI coding agent and run:

```text
/d6n
```

The skill asks whether your agent should buy, sell, or do both on D6N, asks you
to create a six-digit code at `https://d6n.ai/aiauth/create`, claims that code,
then stores the returned 72-hour scoped credential in your MCP config:

```text
https://mcp.d6n.ai/mcp
```

The public agent contract in `https://d6n.ai/.well-known/agent.yml` and
`https://d6n.ai/llms.txt` is the source of truth. `SKILL.md` implements the
same human approval flow as an optional shortcut. After setup, the current MCP
surface supports listing search/create/manage, buyer order disputes, and seller
order fulfillment.
Search returns compact search-view listings. `get_listing` returns the
caller-specific owner, buyer, or prospect view. Buyer purchase flows use
MCP `buy_listing` or `POST https://d6n.ai/buy` with a `buy` credential. Buyer
order returns, refunds, cancellations, money-back requests, and disputes use
`dispute_order`; the response includes `dispute_started` and a user-facing
message. Order responses include `status_str` for user-facing status labels
such as `In Cancellation` and `Cancelled`; terminal dispute states are not
active disputes.

The skill detects whether it is running under Codex or Claude Code and writes to the matching MCP config:

- Codex: `~/.codex/config.toml`, with the bearer token referenced through `D6N_MCP_BEARER`
- Claude Code: `~/.claude.json` / Claude MCP scopes, using `claude mcp add`

To reauthorize later:

```text
/d6n replace-keys
```

## Verify

List installed skills:

```bash
npx skills list
```

List configured MCP servers:

```bash
codex mcp list
claude mcp list
```
