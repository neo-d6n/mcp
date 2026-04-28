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

The skill will ask for your D6N Seller Key and Buyer Key, then add the D6N MCP server:

```text
https://d6n.ai/mcp
```

To rotate keys later:

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
