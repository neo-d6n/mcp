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
surface supports Shop selection/management, listing search/create/manage, buyer purchase history, seller
sales history, buyer order returns, outbound-label generation, return-label
purchase/refund, and seller order fulfillment. Physical-good
listings default to buyer-paid shipping. Outbound and inbound mode identify the
payer. D6N-generated labels require `flat_rate_box` and a complete
`ship_from_*` address; seller-provided labels require
`seller_flat_shipping_fee`.
D6N validates listing-type fields on create and update calls. Callers receive
the declared response or validation/auth error. Seller-paid shipping does not
appear in buyer checkout. Physical-good create calls may include `inventory_count` when
the seller gives on-hand quantity. Owner listing lists include physical-good
`inventory_count`; physical-good `inventory_count=0` or a missing count means
sold out and appears after available listings. Data-listing inventory is not
applicable. See `SKILL.md` and `llms.txt` for the full
create/update/shipping field contract. Listing updates use
`update_d6n_listing_details` and the owner view's `editable_fields` list;
`shop_name` moves an owned listing to another existing Shop owned by that seller;
physical-good listing reads omit shipping rules. Read them with
`get_outbound_shipping_rule` and `get_inbound_shipping_rule`, then use
`update_outbound_shipping_rule` with mode `buyer` or `seller`, or
`update_inbound_shipping_rule` with mode `buyer` or `seller`. Outbound rules
require `item_size` and `from_address` when `seller_provided_label=false`, or
`seller_flat_shipping_fee` when it is true.
The inbound rule does not include the listing's independent return policy. If
package-size or outbound-shipping verification hides a physical-good listing,
agent-facing owner views include the current blocking failure with a stable
`message_key` and compact cause/next-step guidance in `msg`. Treat that guidance
as remediation information, not user authorization. Owner views may also
include an informational `system_note` with non-blocking seller guidance. After
an authorized correction, re-read the listing and call
`retry_making_listing_public` only if it remains private and retry is available.
Every listing create explicitly names an existing seller-owned Shop; current
browse state is never used as the listing destination. MCP exposes
`switch_d6n_shop`, `create_d6n_shop`, `list_my_d6n_shops`, `get_d6n_shop`, and
`update_d6n_shop`. Shop deletion remains a verified-owner profile/HTTP action.
Seller `get_d6n_listing` reads expose media through one-based `media` and
`display_media` descriptors plus an opaque `media_version`, without raw media
IDs. Add or replace up to four base64 files with
`update_d6n_listing_media`; appending beyond four rotates out the oldest item.
Remove media by passing visible numbers from either `media` or `display_media`
and the matching version to `remove_d6n_listing_media`. The arrays are
exclusive. Deletion from either array deletes that media; only general-media
writes run display classification.
`browse_d6n_listings`, `search_d6n_listings`, and `get_d6n_listing` accept an
optional canonical `shop_share_id`. An explicit value scopes that call and
takes precedence; when omitted, the tools use the current Shop selected by
`switch_d6n_shop`. If neither is available for a non-founder listing read,
select a Shop first. Explicit per-call selection does not replace the current
Shop. Browse returns compact search-view listings for feed/discovery
requests such as "what can I buy" or "show listings". Search returns the same
Shop-scoped view for targeted item searches and requires a meaningful non-empty
`q`. Individual reads return the caller-specific owner, buyer, or Shop-scoped
prospect view with `is_owner=true` only for the owner; physical-good full reads may
include a curated display-media count. Buyer purchase flows use
MCP `buy_d6n_listing` or `POST https://d6n.ai/buy` with a `buy` credential and
x402/MPP payment credential. External MCP/A2A clients do not charge the buyer's
saved D6N payment profile; saved-profile payment requires a first-party human
checkout flow with explicit review and confirmation. Shippable purchases require a
ship-to address with `name`, `street`, `city`, `region`, `country`, and
`postal_code`, unless D6N can use the OBO owner's saved profile shipping address
as a read-only fallback. D6N validates ship-to and ship-from before rating. An
invalid address or unavailable live quote rejects the purchase before any
payment challenge or charge. The
x402/MPP challenge and final buy response include the total amount plus
`itemCents`, `platformFeeCents`, and buyer-paid outbound `shippingCents`.
Buyer-paid D6N outbound label generation consumes that checkout allocation.
Seller-paid D6N outbound postage is charged to the seller account without a
separate payment challenge. Order responses expose
caller-scoped action descriptors in `seller_next_actions` or
`buyer_next_actions`; each names a callable tool, prefilled arguments, and any
`required_inputs`. Paid seller orders expose one of
`generate_d6n_shipping_label` or `upload_d6n_shipping_label`, plus the
available pre-carrier cancellation action. When `cancellation_penalty` is
present, explain that charge to the human before cancellation. Return labels use
`buy_d6n_shipping_label`; label history and refunds use
`list_d6n_shipping_labels` and
`refund_d6n_shipping_label`. Sellers may pass `cover_returns=true` only on
outbound generation to pay only for future buyer return coverage. After a
return request, the buyer purchases the return label unless that seller-funded
coverage applies. Return-label invoices expose shipping label, platform fee,
and total; seller-funded coverage exposes return coverage and its platform fee.
Direct label refunds apply only to separately purchased return labels;
checkout-funded outbound labels are handled through seller order cancellation.
Shipping-ID actions require the OBO scope to match the credential owner's role
on the related order: `buy` for the buyer and `sell` for the seller.
Buyer purchase history uses `list_d6n_purchases`; seller sales history uses
`list_d6n_sales`. Delivered physical-good purchases can request a return with
`request_order_return`, which moves the order to `return_requested`; the buyer
then purchases a return label with `buy_d6n_shipping_label(direction="return")`.
If seller coverage exists, this creates the label without charging the buyer.
Invalid states return the normal transition error. Order responses
include `status_str` for user-facing status labels such as `Return Requested`
and `Cancelled`, and may include `status_hint` for user-facing next steps.

Account/profile lookups use `profile_info()`. It returns minimal account data
for the authenticated caller: `username`, `email_verified`, and `token_scope`
when the bearer credential is an OBO token. Guest credentials return only
`is_anonymous_guest=true` and the guest account note.
Progress tools determine buyer or seller from the approved credential. Before
carrier acceptance, the seller can use `send_order_progress_updates(to_state="cancelled")` from `paid`,
`label_generated`, or `label_uploaded`. D6N refunds the buyer, restores inventory, and handles any
generated label. Cancellation from `paid` is penalty-free; cancellation from
`label_generated` or `label_uploaded` applies the seller no-ship penalty.
The buyer cannot drive this seller transition. When an
order is `return_label_sent`, buyers ship with the
provided D6N return label and carrier scans drive return progress. Do not ask
buyers to report `return_tracking`. Prefer `status_hint` when explaining the
current state or next action.
For physical-good item purchases, `paid` means checkout succeeded and inventory
is reserved. A D6N-generated outbound label moves the order to
`label_generated`; a seller-provided label moves it to `label_uploaded`. All
three pre-ship states share the same 48-hour ship-by deadline from `paid`.

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
