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
claim a human-created agent auth code, and configure `https://mcp.d6n.ai/mcp` directly. Use this skill only as
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

Use the D6N MCP endpoint:

- `https://mcp.d6n.ai/mcp`

Use the D6N agent auth HTTP origin:

- `https://d6n.ai`

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

Keep this value short and readable.

### 2. Ask The Human For A D6N Agent Auth Code

Ask the human to open:

```text
https://d6n.ai/aiauth/create
```

Tell them to log in, enter the suggested agent name (`<CLIENT_ID>`), select the requested `buy` and/or `sell` scope, click Create, and return with the six-digit code. If they already have a code, use it.

The owner may click Cancel before creating the code. A created code is valid
for 1 hour and can be claimed once. A successful claim returns a credential
that currently expires after 72 hours. Reauthorizing with the same owner and
agent name replaces older active D6N OBO credentials for that agent name.

### 3. Claim The Code

After the user gives the six-digit code, call:

```bash
curl -sS -X POST "$D6N_HTTP_ORIGIN/aiauth/claim/<CODE>"
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
list_my_d6n_listings(limit=1)
```

For a `buy` credential, prefer:

```text
browse_d6n_listings(limit=1)
```

Expected: the call completes without an auth/configuration error. An empty list
is valid.

## Current MCP Tools

After the server is configured, use the MCP tools exposed by the live D6N
server. The current production surface covers listing search/create/manage,
buyer order returns, and seller order fulfillment.

Account/profile tool:

- `profile_info()`: return minimal D6N account info for the authenticated caller. Guests return `is_anonymous_guest=true` plus a note; authenticated users return `username`, `email_verified`, and `token_scope` when the bearer credential is an OBO token.

Listing creation tools require `sell` scope:

- `create_physical_good_listing(files=None, title=None, description=None, price_usd=None, condition=None, flat_rate_box=None, ship_from_name=None, ship_from_street=None, ship_from_city=None, ship_from_region=None, ship_from_postal_code=None, ship_from_country=None, inventory_count=None)`: create a physical-good listing from the gathered draft. MCP clients provide media as base64 `files`; `media_ids` are not accepted on MCP. Callers receive the declared response or validation/auth error.

Every create-listing call must include `price_usd` as a decimal USD amount,
for example `5.43`, or `0` for a free listing.
For physical goods, include `inventory_count` when the seller gives on-hand
quantity while creating the listing.

Physical goods use D6N-managed shipping in this activation. New physical-good
listings default to `shipping_mode="d6n"` and must pass
`flat_rate_box` (`envelope`, `small`, `medium`, or `large`) plus a ship-from
address (the `ship_from_*` fields). D6N validates listing-type fields and
returns the declared response or validation/auth error.
When a buyer purchases a physical good, the item-purchase challenge and response
charge item + platform fee + outbound `shippingCents`. Sellers later generate
that buyer-paid outbound label without another postage charge. Buyers purchase
return labels after requesting a return, unless the seller funded return
coverage with `cover_returns=True` while generating the outbound label.
MCP/A2A clients provide `shipping_address` up front or use the OBO owner's
saved profile shipping fallback when available.

Newly created listings are public by default and can appear in public
marketplace search.

Listing read/manage tools:

- `browse_d6n_listings(listing_type, tags_any, languages_any, amenities_any, price_cents_min, price_cents_max, currency, category, location_city, location_region, location_country, service_type, sort, limit, cursor)`: public feed/discovery view. Use for "what can I buy", "what is available", or "show listings"; do not pass a natural-language query.
- `search_d6n_listings(q, listing_type, tags_any, languages_any, amenities_any, price_cents_min, price_cents_max, currency, category, location_city, location_region, location_country, service_type, sort, mode, limit, cursor)`: targeted public search view. Use only when the user names or describes an item; `q` must be meaningful and non-empty.
- `list_my_d6n_listings(limit=50)`: owner view for listings created by the authenticated user. Physical-good owner rows include `inventory_count`; physical-good `inventory_count=0` or a missing count means sold out and appears after available listings. Data-listing inventory is not applicable.
- `get_d6n_listing(datum_id)`: owner view for the seller, buyer view for the purchaser, or prospect view for an authenticated non-purchaser on public listings.
- `update_d6n_listing_details(datum_id, fields=None, price_usd=None, open_to_public=None, access_terms=None, product_url=None, seller_notes=None, inventory_count=None, condition=None, flat_rate_box=None, ship_from_name=None, ship_from_street=None, ship_from_city=None, ship_from_region=None, ship_from_postal_code=None, ship_from_country=None, brand=None, model=None, color=None, dimensions=None, weight=None, return_policy=None)`: update editable owner fields; requires `sell` scope and ownership. First read the owner view and use only `editable_fields`. `price_usd` converts to `price_cents`.
- `delete_d6n_listing(datum_id)`: permanently delete a listing owned by the authenticated user; requires `sell` scope and ownership.
- `update_d6n_listing_media(datum_id, files, replace=False)`: append media to a seller-owned listing, or replace the complete media set when `replace=True`; requires `sell` scope and ownership. D6N re-runs extraction and rebuilds physical-good display images from product photos.
- `retry_making_listing_public(datum_id)`: rerun failed D6N listing verifications for a hidden listing and make it public if the failures clear; if D6N says the listing is not ready to retry, edit listing details or media first. Requires `sell` scope and ownership.
- `buy_d6n_listing(datum_id, payment_credential=None, quantity=None, shipping_address=None, booking_start_time=None, booking_end_time=None, params=None)`: purchase a listing with a `buy` credential. External MCP/A2A clients pay with x402/MPP only: call once to receive the challenge, then retry with `payment_credential` after completing the machine-payment path. For shippable listings, pass a deliverable `shipping_address` with `name`, `street`, `city`, `region`, `country`, and `postal_code`; if omitted, D6N may use the OBO owner's saved profile shipping address, and if neither exists the payment attempt is rejected before any charge. D6N validates ship-to and ship-from before rating; an invalid address or unavailable live quote rejects the purchase before any payment challenge or charge. The challenge and final response include the total amount plus `itemCents`, `platformFeeCents`, and buyer-paid checkout `shippingCents`.
- `request_order_return(order_id)`: request a return for a delivered physical-good purchase. It moves the order from `delivered` to `return_requested`; invalid states return the normal transition error. This is distinct from booking cancellation.

Physical-good listing updates have the same D6N-managed shipping rules as
chat.d6n.ai and A2A: `shipping_mode` is not editable in this activation. Use
`flat_rate_box` and the complete `ship_from_*` address to update label
configuration. Package-size verification can hide a listing with owner-only
`hide_reason.fails`; use `retry_making_listing_public` to rerun failed checks.
If D6N says the listing is not ready to retry, edit listing details or
media first.

Order tools:

Shipping-label tools require `buy` or `sell` scope and an order-party match.
Direction `outbound` generates the buyer-checkout-funded seller label;
direction `return` purchases a buyer return label unless seller coverage applies.
`refund_d6n_shipping_label` applies only to separately purchased return labels;
checkout-funded outbound labels are unwound through seller order cancellation.

- `get_d6n_order(order_id)`
- `list_d6n_purchases(limit=20)`
- `list_d6n_sales(limit=20)`
- `request_order_return(order_id)`
- `buy_d6n_shipping_label(order_id, direction, carrier=None, cover_returns=False, payment_credential=None)`
- `list_d6n_shipping_labels(limit=20)`
- `refund_d6n_shipping_label(shipping_id)`
- `get_order_progress_requirements(order_id)`
- `send_order_progress_updates(order_id, to_state, inputs={})`

Order responses expose human-readable UTC time fields ending in `_str`.
Prefer those fields when describing deadlines or order history to a user.
MCP order tools request Unix timestamp fields alongside the `_str` fields for
programmatic use. Order responses include `quantity`, the purchased item count.
Use `status_str` for user-facing status and `status_hint`, when
present, for the next-step explanation.
D6N-managed labels use the shipping-label service. Sellers use
`buy_d6n_shipping_label(order_id, direction="outbound")` for paid orders. The
outbound label uses shipping already paid by the buyer in item checkout and
does not charge the seller again. `cover_returns=True` charges the seller only
for future return-label coverage, never outbound postage. Buyers use
`buy_d6n_shipping_label(order_id, direction="return")` for return-requested
orders and pay for that return label unless seller coverage exists. Progress
tools determine buyer or seller from the approved credential. Before carrier
acceptance, the seller can use
`get_order_progress_requirements` and
`send_order_progress_updates(to_state="cancelled")` from `paid` or
`label_generated`; D6N refunds the buyer, restores inventory, and handles any
generated label.
The buyer cannot drive this seller transition. When an order is
`return_label_sent`, buyers ship with the
provided D6N label and carrier scans
drive return progress. Do not ask buyers to report `return_tracking`. Prefer
`status_hint` when explaining the current state or next action.
For physical-good item purchases, `paid` means checkout succeeded and inventory
is reserved. Generating the outbound label moves the order to
`label_generated`. The `paid` and `label_generated` states share the same
48-hour ship-by deadline from `paid`.

D6N physical-good item checkout always includes outbound shipping in the
buyer's invoice. MCP `buy_d6n_listing` and `POST https://d6n.ai/buy` complete
that checkout. MCP `buy_d6n_shipping_label(direction="outbound")` and
`POST https://d6n.ai/buy/shipping` generate the seller's already-paid outbound
label. Direction `return` purchases buyer return postage after a return request,
unless seller-funded coverage applies. External MCP/A2A payment credentials are
used for item checkout, buyer return labels, and optional seller-funded return
coverage; they are not used to charge outbound postage a second time.

Use `datum_id` as the listing identifier. For create tools, `files` is required
for every listing type and
should contain one or more objects like
`{"filename":"example.pdf","file_base64":"..."}`. Do not use the old `session`
or open-order tool names unless the server explicitly exposes them in the
active MCP tool list.

Listing responses are intentionally read-mode specific. Public search returns
the compact search view. `get_d6n_listing` returns a prospect, buyer, or owner view
based on the authenticated caller. Owner views include `editable_fields`.
Prospect views may include `search_fields`, a list of field names useful for a
compact display digest; the values are already present on the payload.
Physical-good `get_d6n_listing` owner/buyer/prospect reads may include
`display_image`, the curated product photos for the listing. Search omits
attachments. Use only fields returned by the selected listing view.

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
