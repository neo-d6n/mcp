# D6N MCP Install Skill

This repository contains the `d6n` skill for installing and configuring the D6N MCP tools.

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

The skill asks whether your agent should buy, sell, or do both on D6N, creates an OAuth consent URL for that role, asks you to approve it in the browser, then stores the returned OAuth credential in your MCP config:

```text
https://d6n.ai/mcp
```

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
claude mcp list
```
