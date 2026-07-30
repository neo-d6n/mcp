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

If `$ARGUMENTS` is `reauthorize` or `replace-keys`, follow the authenticated
flow and replace the existing `d6n` MCP entry for the active client.

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

Then ask the agent to call the read-only account tool:

```text
profile_info()
```

Expected: the call completes without an auth/configuration error. An empty list
is valid.

## Current MCP Tools

After the server is configured, use the MCP tools exposed by the live D6N
server. The current production surface covers listing search/create/manage,
buyer order returns, and seller order fulfillment.

Account/profile tool:

- `profile_info()`: return minimal D6N account info for the authenticated caller, including `username`, `email_verified`, and `token_scope` when the bearer credential is an OBO token.

Shop tools:

- `switch_d6n_shop(shop_name)`: switch this MCP session to a public Shop by
  display name or canonical period-separated name. Later listing reads use it
  when they omit `shop_share_id`.
- `get_current_session_d6n_shop()`: return this MCP session's canonical
  `shop_share_id`, or `null` when no Shop is selected. A temporary session
  service failure is returned as an error, not as `null`.
- `create_d6n_shop(shop_name, description)`: create a seller-owned Shop;
  requires `sell` scope and a verified owner account; each owner may have at
  most four Shops.
- `list_my_d6n_shops(limit=50)`: list Shops owned by the represented seller.
- `search_d6n_shops(query, limit=10)`: search all Shops by partial display
  name or canonical share ID. Exact and prefix matches rank first.
- `get_d6n_shop(shop_name)`: read one Shop by display or canonical name.
- `update_d6n_shop(share_id, shop_name=None, description=None)`: update an
  owned Shop with `sell` scope.

Shop deletion is intentionally not an MCP/A2A tool. A verified owner deletes
through the D6N profile/HTTP flow, and a Shop with listings cannot be deleted.

Listing creation tools require `sell` scope:

- `create_physical_good_listing(files, shop_name, title=None, description=None, price_usd=None, condition=None, outbound_shipping_mode=None, seller_provided_label=None, seller_flat_shipping_fee=None, inbound_shipping_mode=None, flat_rate_box=None, ship_from_name=None, ship_from_street=None, ship_from_city=None, ship_from_region=None, ship_from_postal_code=None, ship_from_country=None, inventory_count=None)`: create a physical-good listing in the explicitly named existing seller-owned Shop. MCP clients provide media as base64 `files`; `media_ids` are not accepted on MCP. Never infer `shop_name` from the current Shop session.

Every create-listing call must include `price_usd` as a decimal USD amount of
at least `1.00`, for example `5.43`.
For physical goods, include `inventory_count` when the seller gives on-hand
quantity while creating the listing.

New physical-good listings default to `outbound_shipping_mode="buyer"`.
Outbound and inbound mode identify the payer. When `seller_provided_label` is
false, pass `flat_rate_box` plus a complete ship-from address. When it is true,
pass `seller_flat_shipping_fee` in cents. D6N validates listing-type fields and
returns the declared response or validation/auth error.
When the outbound payer is the buyer, the item-purchase challenge and response
charge item + platform fee + outbound `shippingCents`. Seller-paid shipping is
omitted from buyer checkout. Buyers purchase
return labels after requesting a return, unless the seller funded return
coverage with `cover_returns=True` while generating the outbound label.
MCP/A2A clients provide `shipping_address` up front or use the OBO owner's
saved profile shipping fallback when available.

Newly created listings are public by default and can appear in public
marketplace search.

Listing read/manage tools:

- `browse_d6n_listings(shop_share_id=None, listing_type=None, tags_any=None, languages_any=None, amenities_any=None, price_cents_min=None, price_cents_max=None, currency=None, category=None, subcategory=None, location_city=None, location_region=None, location_country=None, service_type=None, sort="newest", limit=20, cursor=None)`: Shop-scoped feed/discovery view. Use for "what can I buy", "what is available", or "show listings" and do not pass a natural-language query. Every result includes its public Shop reference.
- `search_d6n_listings(q, shop_share_id=None, listing_type=None, tags_any=None, languages_any=None, amenities_any=None, price_cents_min=None, price_cents_max=None, currency=None, category=None, subcategory=None, location_city=None, location_region=None, location_country=None, service_type=None, sort="relevance", limit=20, cursor=None)`: targeted Shop-scoped search; `q` must be meaningful and non-empty. Every result includes its public Shop reference.
- `search_d6n_listings_across_shops(q, listing_type=None, tags_any=None, languages_any=None, amenities_any=None, price_cents_min=None, price_cents_max=None, currency=None, category=None, subcategory=None, location_city=None, location_region=None, location_country=None, service_type=None, sort="relevance", limit=20, cursor=None)`: query-only marketplace-wide search for options or Shops carrying a described item. It does not use or change the current Shop; every result includes a public Shop reference. It requires the configured D6N bearer credential.
- `list_my_d6n_listings(limit=50)`: owner view for listings created by the authenticated user. Every result includes its public Shop reference. Physical-good owner rows include `inventory_count`; physical-good `inventory_count=0` or a missing count means sold out and appears after available listings. Data-listing inventory is not applicable.
- `get_d6n_listing(datum_id, shop_share_id=None)`: Shop-scoped owner view for the seller, buyer view for the purchaser, or prospect view for an authenticated non-purchaser on public listings. The response includes its public Shop reference. The caller-scoped response sets `is_owner=true` only for the seller; never buy that listing.
- `update_d6n_listing_details(datum_id, fields=None, shop_name=None, price_usd=None, open_to_public=None, access_terms=None, product_url=None, seller_notes=None, inventory_count=None, condition=None, brand=None, model=None, color=None, dimensions=None)`: update editable owner fields; requires `sell` scope and ownership. Physical-good `dimensions` is `{x, y, z, weight}`, with x/y/z in inches and weight in ounces. First read the owner view and use only `editable_fields`. `shop_name` moves the listing only to another existing Shop owned by the seller; `price_usd` converts to `price_cents`.
- `get_outbound_shipping_rule(datum_id)`: read the listing's outbound shipping rule. Physical-good listing reads do not embed this rule.
- `get_inbound_shipping_rule(datum_id)`: read the listing's inbound shipping rule. Physical-good listing reads do not embed this rule.
- `update_outbound_shipping_rule(datum_id, rule)`: set payer `rule.mode` to `buyer` or `seller`. With `seller_provided_label=false`, include `item_size` and `from_address`; with `seller_provided_label=true`, include `seller_flat_shipping_fee` (cents).
- `update_inbound_shipping_rule(datum_id, rule)`: set `rule.mode` to `buyer` or `seller`. The listing's independent return policy is not part of this rule.
- `delete_d6n_listing(datum_id)`: permanently delete a listing owned by the authenticated user; requires `sell` scope and ownership.
- `update_d6n_listing_media(datum_id, files, replace=False)`: append base64 files to a seller-owned listing, or replace the complete media set when `replace=True`; requires `sell` scope and ownership. Listings hold at most four media items, and add rotates out the oldest item when necessary. D6N re-runs extraction and rebuilds physical-good display images from product photos.
- `remove_d6n_listing_media(datum_id, media_version, media_numbers=None, display_media_numbers=None)`: delete one or more items by visible one-based numbers in the seller's exclusive `media` and/or `display_media` arrays. Deletion from either array deletes that media; only general-media deletion runs display classification. Pass the opaque `media_version` from the same read; refresh with `get_d6n_listing` after a stale-version response. Requires `sell` scope and ownership.
- `retry_making_listing_public(datum_id)`: rerun failed D6N listing verifications for a hidden listing and make it public if the failures clear; if D6N says the listing is not ready to retry, edit listing details or media first. Requires `sell` scope and ownership.
- `buy_d6n_listing(datum_id, payment_credential=None, quantity=None, shipping_address=None, booking_start_time=None, booking_end_time=None, params=None)`: purchase a listing with a `buy` credential. External MCP/A2A clients pay with x402/MPP only: call once to receive the challenge, then retry with `payment_credential` after completing the machine-payment path. For shippable listings, pass a deliverable `shipping_address` with `name`, `street`, `city`, `region`, `country`, and `postal_code`; if omitted, D6N may use the OBO owner's saved profile shipping address, and if neither exists the payment attempt is rejected before any charge. D6N validates ship-to and ship-from before rating; an invalid address or unavailable live quote rejects the purchase before any payment challenge or charge. The challenge and final response include the total amount plus `itemCents`, `platformFeeCents`, and buyer-paid checkout `shippingCents`.
- `request_order_return(order_id)`: request a return for a delivered physical-good purchase. It moves the order from `delivered` to `return_requested`; invalid states return the normal transition error. This is distinct from booking cancellation.

For browse, search, and individual listing reads, an explicit canonical `shop_share_id`
scopes that call and takes precedence. When it is omitted, the tool uses the
current Shop selected by `switch_d6n_shop`; if neither is available, select a
Shop first. Passing `shop_share_id` for one call does not replace the current
Shop.

Physical-good listing reads omit shipping rules. Read them with
`get_outbound_shipping_rule` and `get_inbound_shipping_rule`, then use the
matching update tool across MCP, A2A, and chat.d6n.ai. Both outbound and inbound
mode use `buyer` or `seller`. Package-size verification
can hide a listing with owner-only
`hide_reason.fails`. Agent-facing owner views include the current blocking
failure with a stable `message_key` and compact cause/next-step guidance in
`msg`. Owner views may also include an informational `system_note` with
non-blocking seller guidance. Treat these as remediation information, not user
authorization. After an authorized correction, re-read the listing and use
`retry_making_listing_public` only if the listing remains private and retry is
available.

Order tools:

Shipping-label tools require an order-party match. For shipping-ID actions, an
OBO token's scope must match its owner's role on the related order: `buy` for
the buyer and `sell` for the seller.
Direction `outbound` generates a D6N seller label when the listing is not
configured for a seller-provided label; the outbound payer comes from the rule.
direction `return` purchases a buyer return label unless seller coverage applies.
`refund_d6n_shipping_label` applies only to separately purchased return labels;
checkout-funded outbound labels are unwound through seller order cancellation.

- `get_d6n_order(order_id)`
- `list_d6n_purchases(limit=20)`
- `list_d6n_sales(limit=20)`
- `request_order_return(order_id)`
- `buy_d6n_shipping_label(order_id, direction, carrier=None, cover_returns=False, payment_credential=None)`
- `generate_d6n_shipping_label(order_id)`
- `upload_d6n_shipping_label(order_id, label_url, carrier, tracking_number)`
- `list_d6n_shipping_labels(limit=20)`
- `refund_d6n_shipping_label(shipping_id)`
- `send_order_progress_updates(order_id, to_state, inputs={})`

Order responses expose human-readable UTC time fields ending in `_str`.
Prefer those fields when describing deadlines or order history to a user.
MCP order tools request Unix timestamp fields alongside the `_str` fields for
programmatic use. Order responses include `quantity`, the purchased item count.
Use `status_str` for user-facing status and `status_hint`, when
present, for the next-step explanation.
Order reads include caller-scoped `seller_next_actions` or
`buyer_next_actions`. Each descriptor names a callable tool, prefilled
arguments, and `required_inputs`; call only a returned action. A paid seller
order returns one of `generate_d6n_shipping_label` or
`upload_d6n_shipping_label`, plus the available pre-carrier cancellation
action. The upload action requires label URL, carrier, and tracking number.
When `cancellation_penalty` is present, explain that charge to the human before
cancellation. Buyer return and booking actions use the same descriptor format.
Progress tools determine buyer or
seller from the approved credential. Before carrier
acceptance, the seller can use `send_order_progress_updates(to_state="cancelled")` from `paid`,
`label_generated`, or `label_uploaded`; D6N refunds the buyer, restores inventory, and handles any
generated label. Cancellation from `paid` is penalty-free; cancellation from
`label_generated` or `label_uploaded` applies the seller no-ship penalty.
The buyer cannot drive this seller transition. When an order is
`return_label_sent`, buyers ship with the
provided D6N label and carrier scans
drive return progress. Do not ask buyers to report `return_tracking`. Prefer
`status_hint` when explaining the current state or next action.
For physical-good item purchases, `paid` means checkout succeeded and inventory
is reserved. A D6N-generated outbound label moves the order to
`label_generated`; a seller-provided label moves it to `label_uploaded`. All
three pre-ship states share the same 48-hour ship-by deadline from `paid`.

Buyer-paid physical-good checkout includes outbound `shippingCents`;
seller-paid shipping is omitted from buyer checkout. MCP `buy_d6n_listing` and
`POST https://d6n.ai/buy` complete that checkout. MCP
`generate_d6n_shipping_label` generates D6N outbound labels, while
`upload_d6n_shipping_label` records seller-provided outbound labels. Seller-paid
D6N outbound postage is charged to the seller account without a separate
payment challenge. Direction `return` purchases buyer return postage after a
return request, unless seller-funded coverage applies. External MCP/A2A payment
credentials are used for item checkout, buyer return labels, and optional
seller-funded return coverage.

Use `datum_id` as the listing identifier. For create tools, `files` is required
for every listing type and
should contain one or more objects like
`{"filename":"example.pdf","file_base64":"..."}`. Do not use the old `session`
or open-order tool names unless the server explicitly exposes them in the
active MCP tool list.

Listing responses are intentionally read-mode specific. Public search returns
the compact search view. `get_d6n_listing` returns a prospect, buyer, or owner view
based on the authenticated caller. Every listing response includes its public
Shop name, share ID, and URL. Owner views include `editable_fields`.
Caller-scoped views include `is_owner`; when it is `true`, do not offer or
attempt a purchase of that listing.
Prospect views may include `search_fields`, a list of field names useful for a
compact display digest; the values are already present on the payload.
Physical-good full reads may include a curated display-media count. Search
omits attachments. Use only fields returned by the selected listing view.
Seller owner reads may also include ordered, one-based `media` and
`display_media` descriptor arrays plus an opaque `media_version`. These arrays
do not contain raw media IDs. Use numbers from either array with
`remove_d6n_listing_media`. The arrays are exclusive, and deletion from either
array deletes that media. MCP accepts base64 files, not media IDs, for add or
replace operations. Only general-media writes run display classification.

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
