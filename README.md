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
surface supports listing search/create/manage, buyer purchase history, seller
sales history, buyer order returns, and seller order fulfillment. Physical-good
listings use D6N-managed shipping in this activation: create calls default to
`shipping_mode=d6n`, require `flat_rate_box` and a complete `ship_from_*`
address, and charge the buyer item + flat-rate shipping so D6N can buy the
carrier label. Physical-good create calls may include `inventory_count` when
the seller gives on-hand quantity. See `SKILL.md` and `llms.txt` for the full
create/update/shipping field contract. Listing updates use
`update_d6n_listing_details` and the owner view's `editable_fields` list;
`shipping_mode` is not editable in this activation.
Search returns compact search-view listings. `get_d6n_listing` returns the
caller-specific owner, buyer, or prospect view; physical-good full reads may
include curated `display_image` product media IDs. Buyer purchase flows use
MCP `buy_d6n_listing` or `POST https://d6n.ai/buy` with a `buy` credential and
x402/MPP payment credential. External MCP/A2A clients do not charge the buyer's
saved D6N payment profile; saved-card profile payment is only for first-party
browser UI after explicit human confirmation. Shippable purchases require a
ship-to address with `name`, `street`, `city`, `region`, `country`, and
`postal_code`, unless D6N can use the OBO owner's saved profile shipping address
as a read-only fallback. The x402/MPP challenge and final buy response include
the shipping-inclusive amount plus `itemCents` and `shippingCents`; MCP/A2A
clients do not run the browser-only shipping-estimate confirmation step.
Buyer purchase history uses `list_d6n_purchases`; seller sales history uses
`list_d6n_sales`. Delivered physical-good purchases can request a return with
`request_order_return`, which moves the order to `return_requested`; D6N then
buys the return label and moves the order to `return_label_sent` when Shippo
succeeds. Invalid states return the normal transition error. Order responses
include `status_str` for user-facing status labels such as `Return Requested`
and `Cancelled`. Progress tools infer buyer or seller from the authenticated
token owner user id on the order. When an order is `return_label_sent`, buyer
agents use `send_order_progress_updates` with `to_state=return_shipped` plus
`return_tracking`. For physical goods, `return_shipped` and `return_to_sender`
trigger the full buyer refund while processor fees and bought Shippo labels
remain platform losses. D6N polls `shipped`/`in_transit` and
`return_shipped`/`return_in_transit` tracking on each SLA tick;
`return_delivered` and `return_delivery_failed` collapse the order to
`returned` with a note.

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
