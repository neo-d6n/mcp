---
name: shopmine
description: Optional shortcut to install or reauthorize the ShopMine MCP server for the active AI coding agent using a human-created ShopMine agent auth code.
argument-hint: [reauthorize|replace-keys]
allowed-tools: Bash, AskUserQuestion
---

<!--
Copyright 2026 Neosphere Inc
SPDX-License-Identifier: Apache-2.0
-->

# ShopMine MCP Install Shortcut

This skill is optional. ShopMine does not require skill installation: agents can
discover `https://shopmine.ai/.well-known/agent.yml` or `https://shopmine.ai/llms.txt`,
claim a human-created agent auth code, and configure `https://mcp.shopmine.ai/mcp` directly. Use this skill only as
a convenience installer or reauthorization helper for clients that support
skills. After install, the ShopMine MCP server itself exposes the working tools.

ShopMine grants access through a human-created six-digit agent auth code. The
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

Use the ShopMine MCP endpoint:

- `https://mcp.shopmine.ai/mcp`

Use the ShopMine agent auth HTTP origin:

- `https://shopmine.ai`

## Install Flow

If `$ARGUMENTS` is `reauthorize` or `replace-keys`, follow the authenticated
flow and replace the existing `shopmine` MCP entry for the active client.

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

### 2. Ask The Human For A ShopMine Agent Auth Code

Ask the human to open:

```text
https://shopmine.ai/aiauth/create
```

Tell them to log in, enter the suggested agent name (`<CLIENT_ID>`), select the requested `buy` and/or `sell` scope, click Create, and return with the six-digit code. If they already have a code, use it.

The owner may click Cancel before creating the code. A created code is valid
for 1 hour and can be claimed once. A successful claim returns a credential
that currently expires after 72 hours. Reauthorizing with the same owner and
agent name replaces older active ShopMine OBO credentials for that agent name.

### 3. Claim The Code

Do not run a standalone claim request. Its success response contains the secret
credential. Instead, use the single claim-and-configure block in the next step
so the response is captured in a private temporary file and the credential
never appears in terminal or tool output.

Results:

- `404`: the code expired, was mistyped, or was already consumed. Ask the human to create a new code.
- Approved result: contains `auth_key`, `client_id`, `scopes`, and `expiration_time`. Use `auth_key` immediately. The result is one-shot and the credential expires after 72 hours.

Never print the full credential.

### 4. Claim And Configure MCP Without Echoing The Credential

Set `ACTIVE_CLIENT` from the detection step, replace `<CODE>` with the safe
six-digit code, and run this entire block as one shell action. For Claude Code,
ask whether to use `local` (this project) or `user` (all projects) and set
`CLAUDE_SCOPE` accordingly. Do not split the claim from configuration, enable
shell tracing, print the response file, or print `CLI_KEY`.

```bash
ACTIVE_CLIENT='<codex-or-claude>'
CLAUDE_SCOPE='<local-or-user>'
CLAIM_CODE='<CODE>'

umask 077
claim_file="$(mktemp)"
chmod 600 "$claim_file"
trap 'unset CLI_KEY; rm -f "$claim_file"' EXIT

claim_status="$(curl -sS -o "$claim_file" -w '%{http_code}' -X POST \
  "https://shopmine.ai/aiauth/claim/${CLAIM_CODE}")"
if [ "$claim_status" != '200' ]; then
  printf 'ShopMine authorization failed (HTTP %s). Create a new code and retry.\n' "$claim_status" >&2
  exit 1
fi

CLI_KEY="$(python3 - "$claim_file" <<'PY'
import json
import re
import sys

try:
    with open(sys.argv[1], encoding="utf-8") as response_file:
        key = json.load(response_file).get("auth_key", "")
except (OSError, ValueError, AttributeError) as exc:
    raise SystemExit("ShopMine returned an invalid authorization response") from exc

if not isinstance(key, str) or not re.fullmatch(r"cli_ke[A-Za-z0-9_-]+", key):
    raise SystemExit("ShopMine authorization response did not contain a valid credential")
sys.stdout.write(key)
PY
)"
rm -f "$claim_file"

case "$ACTIVE_CLIENT" in
  codex)
    mkdir -p "$HOME/.config/shopmine"
    chmod 700 "$HOME/.config/shopmine"
    printf "export SHOPMINE_MCP_BEARER='%s'\n" "$CLI_KEY" > "$HOME/.config/shopmine/mcp.env"
    chmod 600 "$HOME/.config/shopmine/mcp.env"

    PROFILE="$HOME/.zshrc"
    if [ -n "${BASH_VERSION:-}" ]; then PROFILE="$HOME/.bashrc"; fi
    if ! grep -q 'ShopMine MCP bearer for Codex' "$PROFILE" 2>/dev/null; then
      printf '\n# ShopMine MCP bearer for Codex\nsource "$HOME/.config/shopmine/mcp.env"\n' >> "$PROFILE"
    fi

    . "$HOME/.config/shopmine/mcp.env"
    codex mcp remove shopmine 2>/dev/null || true
    codex mcp add shopmine --url "https://mcp.shopmine.ai/mcp" \
      --bearer-token-env-var SHOPMINE_MCP_BEARER
    ;;
  claude)
    case "$CLAUDE_SCOPE" in local|user) ;; *) printf 'Choose Claude scope local or user.\n' >&2; exit 1 ;; esac
    claude mcp remove shopmine --scope "$CLAUDE_SCOPE" 2>/dev/null || true
    claude mcp add --transport http --scope "$CLAUDE_SCOPE" \
      --header "Authorization: Bearer ${CLI_KEY}" \
      shopmine "https://mcp.shopmine.ai/mcp"
    ;;
  *)
    printf 'Choose ACTIVE_CLIENT=codex or ACTIVE_CLIENT=claude.\n' >&2
    exit 1
    ;;
esac

unset CLI_KEY
printf 'ShopMine MCP credential configured; secret value was not printed.\n'
```

Codex stores the bearer in `~/.config/shopmine/mcp.env` with mode `0600` and
references it through an environment variable. Claude Code stores the header
in its selected private MCP configuration. Never use Claude project scope when
that would write a credential to a shareable `.mcp.json`.

### 5. Verify

The user must restart the active AI client before new MCP tools appear.

After restart:

```bash
# Codex
. "$HOME/.config/shopmine/mcp.env" 2>/dev/null || true
codex mcp list
codex mcp get shopmine

# Claude Code
claude mcp list
claude mcp get shopmine
```

Then ask the agent to call the read-only account tool:

```text
profile_info()
```

Expected: the call completes without an auth/configuration error. An empty list
is valid.

## Current MCP Tools

After the server is configured, use the MCP tools exposed by the live ShopMine
server. The marketplace tool contract covers listing search/create/manage,
carrier-order returns, and seller fulfillment by shipping or local pickup.

Account/profile tool:

- `profile_info()`: return minimal ShopMine account info for the authenticated caller, including `username`, `email_verified`, and `token_scope` when the bearer credential is an OBO token.

Shop tools:

- `switch_shopmine_shop(shop_name)`: switch this MCP session to a public Shop by
  display name or canonical period-separated name. Later listing reads use it
  when they omit `shop_share_id`.
- `get_current_session_shopmine_shop()`: return this MCP session's canonical
  `shop_share_id`, or `null` when no Shop is selected. A temporary session
  service failure is returned as an error, not as `null`.
- `create_shopmine_shop(shop_name, description)`: create a seller-owned Shop;
  requires `sell` scope and a verified owner account; each owner may have at
  most four Shops.
- `list_my_shopmine_shops(limit=50)`: list Shops owned by the represented seller.
- `browse_shopmine_shops(limit=50)`: browse recommended public ShopMine Shops and their
  public Shop identity.
- `search_shopmine_shops(query=None, limit=10)`: search all Shops by partial display
  name or canonical share ID. Exact and prefix matches rank first. An omitted
  or blank query returns recommended public ShopMine Shops. Excessive calls may be
  throttled.
- `get_shopmine_shop(shop_name)`: read one Shop by display or canonical name.
- `update_shopmine_shop(share_id, shop_name=None, description=None)`: update an
  owned Shop with `sell` scope.

Shop deletion is intentionally not an MCP/A2A tool. A verified owner deletes
through the ShopMine profile/HTTP flow, and a Shop with listings cannot be deleted.

Listing creation tools require `sell` scope:

- `create_physical_good_listing(files, shop_name, title, description, price_usd, condition, outbound_shipping_mode=None, seller_provided_label=None, seller_flat_shipping_fee=None, inbound_shipping_mode=None, allows_returns=None, flat_rate_box=None, ship_from_name=None, ship_from_street=None, ship_from_city=None, ship_from_region=None, ship_from_postal_code=None, ship_from_country=None, inventory_count=None, local_pickup_enabled=None, local_pickup_postal_code=None, local_pickup_phone=None, local_pickup_buyer_notes=None)`: create a physical-good listing in the explicitly named existing seller-owned Shop. MCP clients provide media as base64 `files`; `media_ids` are not accepted on MCP. Never infer `shop_name` from the current Shop session.

Every create-listing call must include `price_usd` as a decimal USD amount of
at least `1.00`, for example `5.43`.
For physical goods, include `inventory_count` when the seller gives on-hand
quantity while creating the listing.

New physical-good listings default to `outbound_shipping_mode="buyer"` and
`allows_returns=true`. Set `allows_returns=false` only when the seller chooses
not to accept carrier returns. Hand-to-hand returns remain unavailable regardless.
Outbound and inbound mode identify the payer. When `seller_provided_label` is
false, pass `flat_rate_box` plus a complete ship-from address. When it is true,
pass `seller_flat_shipping_fee` in cents, plus `flat_rate_box` and the complete
`ship_from_*` address when returns are allowed. This is also the return address.
ShopMine validates listing-type fields and
returns the declared response or validation/auth error.
For carrier fulfillment when the outbound payer is the buyer, the item-purchase challenge and response
charge item + platform fee + outbound `shippingCents`. Seller-paid shipping is
omitted from buyer checkout. Buyers purchase
return labels after requesting a return, unless the seller funded return
coverage with `cover_returns=True` while generating the outbound label.
MCP/A2A carrier buyers provide `shipping_address` up front or use the OBO owner's
saved profile shipping fallback when available.

Sellers may additionally offer local pickup with `local_pickup_enabled=true`,
`local_pickup_postal_code`, `local_pickup_phone` (international format, e.g. +14155550123), and
`local_pickup_buyer_notes`. Availability and area are public; pickup notes are
shared with the purchaser only through the order after it becomes `paid`.

Newly created listings are public by default and can appear in public
marketplace search.

Listing read/manage tools:

- `browse_shopmine_listings(shop_share_id=None, listing_type=None, tags_any=None, languages_any=None, amenities_any=None, price_cents_min=None, price_cents_max=None, currency=None, category=None, subcategory=None, location_city=None, location_region=None, location_country=None, service_type=None, sort="newest", limit=20, cursor=None)`: Shop-scoped feed/discovery view. Use for "what can I buy", "what is available", or "show listings" and do not pass a natural-language query. Every result includes its public Shop reference.
- `search_shopmine_listings(q, shop_share_id=None, listing_type=None, tags_any=None, languages_any=None, amenities_any=None, price_cents_min=None, price_cents_max=None, currency=None, category=None, subcategory=None, location_city=None, location_region=None, location_country=None, service_type=None, sort="relevance", limit=20, cursor=None)`: targeted Shop-scoped search; `q` must be meaningful and non-empty. Every result includes its public Shop reference.
- `search_shopmine_listings_across_shops(q, listing_type=None, tags_any=None, languages_any=None, amenities_any=None, price_cents_min=None, price_cents_max=None, currency=None, category=None, subcategory=None, location_city=None, location_region=None, location_country=None, service_type=None, sort="relevance", limit=20, cursor=None)`: query-only marketplace-wide search for options or Shops carrying a described item. If the user has not described what they want, explain that cross-Shop search is available and ask what kind of item they are looking for instead of calling the tool; generic nouns such as listings, items, products, or goods are not a search subject. It does not use or change the current Shop; every result includes a public Shop reference. It requires the configured ShopMine bearer credential.
- `list_my_shopmine_listings(limit=50)`: owner view for listings created by the authenticated user. Every result includes its public Shop reference. Physical-good owner rows include `inventory_count`; physical-good `inventory_count=0` or a missing count means sold out and appears after available listings. Data-listing inventory is not applicable.
- `get_shopmine_listing(datum_id, shop_share_id=None)`: caller-scoped owner view for the seller, buyer view for the purchaser, or prospect view for an authenticated non-purchaser on public listings. The response includes its public Shop reference. The caller-scoped response sets `is_owner=true` only for the seller; never buy that listing.
- `update_shopmine_listing_details(datum_id, fields=None, shop_name=None, price_usd=None, open_to_public=None, allows_returns=None, access_terms=None, product_url=None, seller_notes=None, inventory_count=None, condition=None, brand=None, model=None, color=None, dimensions=None)`: update editable owner fields; requires `sell` scope and ownership. Physical-good `dimensions` is `{x, y, z, weight}`, with x/y/z in inches and weight in ounces. First read the owner view and use only `editable_fields`. `shop_name` moves the listing only to another existing Shop owned by the seller; `price_usd` converts to `price_cents`. Omitted `allows_returns` preserves current permission.
- `get_outbound_shipping_rule(datum_id)`: read a seller-owned listing's outbound shipping and local-pickup settings. Physical-good listing reads do not embed the full rule.
- `get_inbound_shipping_rule(datum_id)`: read the listing's return-shipping payer and `allows_returns` permission. Physical-good listing reads do not embed the full rule.
- `update_outbound_shipping_rule(datum_id, rule)`: replace the full outbound rule, including pickup settings. Set payer `rule.mode` to `buyer` or `seller`. With `seller_provided_label=false`, include `item_size` and `from_address`; with `seller_provided_label=true`, include `seller_flat_shipping_fee` (cents) and also `item_size` and `from_address` when returns are allowed. Read the current rule first and resend `local_pickup_enabled=true`, `local_pickup_postal_code`, `local_pickup_phone`, and `local_pickup_buyer_notes` to retain pickup. Omitting `local_pickup_enabled` disables pickup and clears its details. `allows_returns` belongs to the inbound rule, not this rule.
- `update_inbound_shipping_rule(datum_id, rule)`: set `rule.mode` to `buyer` or `seller`, and optionally set boolean `rule.allows_returns`. Omission preserves current return permission; explicit `false` disables carrier returns for future purchases. Configure outbound `item_size` and `from_address` before enabling returns. Disable inbound return permission before removing those outbound details. The editable `allows_returns` field may also be changed with `update_shopmine_listing_details`.
- `delete_shopmine_listing(datum_id)`: permanently delete a listing owned by the authenticated user; requires `sell` scope and ownership.
- `update_shopmine_listing_media(datum_id, files, replace=False)`: append base64 files to a seller-owned listing, or replace the complete media set when `replace=True`; requires `sell` scope and ownership. Listings hold at most four media items, and add rotates out the oldest item when necessary. ShopMine re-runs extraction and rebuilds physical-good display images from product photos.
- `remove_shopmine_listing_media(datum_id, media_version, media_numbers=None, display_media_numbers=None)`: delete one or more items by visible one-based numbers in the seller's exclusive `media` and/or `display_media` arrays. Deletion from either array deletes that media; only general-media deletion runs display classification. Pass the opaque `media_version` from the same read; refresh with `get_shopmine_listing` after a stale-version response. Requires `sell` scope and ownership.
- `retry_making_listing_public(datum_id)`: rerun failed ShopMine listing verifications for a hidden listing and make it public if the failures clear; if ShopMine says the listing is not ready to retry, edit listing details or media first. Requires `sell` scope and ownership.
- `buy_shopmine_listing(datum_id, payment_credential=None, quantity=None, shipping_address=None, fulfillment_mode=None, booking_start_time=None, booking_end_time=None, params=None)`: purchase a listing with a `buy` credential. `fulfillment_mode` accepts `carrier` or `hand-to-hand`; carrier is the physical-good default. Select hand-to-hand only when the listing has `local_pickup_enabled=true`. External MCP/A2A clients pay with x402/MPP only: call once to receive the challenge, then retry with the same purchase arguments and `payment_credential`. Changing fulfillment mode requires a fresh challenge. Pickup needs no shipping address; its challenge returns `fulfillmentMode="hand-to-hand"`, `itemCents`, `platformFeeCents`, and `shippingCents=0`. For carrier fulfillment, pass a deliverable `shipping_address` with `name`, `street`, `city`, `region`, `country`, and `postal_code`; if omitted, ShopMine may use the OBO owner's saved profile shipping address, and if neither exists the payment attempt is rejected before any charge. An invalid carrier address or unavailable shipping quote rejects the purchase before any payment challenge or charge. Successful checkout returns `payment_receipt.orderId`; use it as `order_id` for order follow-up.
- `request_order_return(order_id)`: request a return for a delivered carrier-shipped physical-good purchase with `allows_returns=true`, only when offered in `buyer_next_actions`. It moves the order from `delivered` to `return_requested`; invalid states return the normal transition error. Hand-to-hand orders cannot use this carrier-return workflow.

For browse, search, and individual listing reads, an explicit canonical `shop_share_id`
scopes that call and takes precedence. When it is omitted, the tool uses the
current Shop selected by `switch_shopmine_shop` when available. Browse and
Shop-targeted search require one of those Shop scopes. Passing `shop_share_id`
for one call does not replace the current Shop.

Physical-good search, prospect, and buyer listing views expose
`allows_returns` and `local_pickup_enabled`, plus `local_pickup_postal_code`
when configured. They never include `local_pickup_phone` or
`local_pickup_buyer_notes`; read the paid order for private pickup coordination.
Tell the buyer before purchase when `allows_returns=false` (returns not accepted).

Physical-good listing reads omit full shipping rules. Sellers read them with
`get_outbound_shipping_rule` and `get_inbound_shipping_rule`, then use the
matching update tool across MCP, A2A, and chat.shopmine.ai. Both outbound and inbound
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

Shipping-label tools apply only to carrier orders and require an order-party match. For shipping-ID actions, an
OBO token's scope must match its owner's role on the related order: `buy` for
the buyer and `sell` for the seller.
Direction `outbound` generates a ShopMine seller label when the listing is not
configured for a seller-provided label; the outbound payer comes from the rule.
direction `return` purchases a buyer return label unless seller coverage applies.
`refund_shopmine_shipping_label` applies only to separately purchased return labels;
checkout-funded outbound labels are unwound through seller order cancellation.

- `get_shopmine_order(order_id)`
- `list_shopmine_purchases(limit=20)`
- `list_shopmine_sales(limit=20)`
- `request_order_return(order_id)`
- `buy_shopmine_shipping_label(order_id, direction, carrier=None, cover_returns=False, payment_credential=None)`
- `generate_shopmine_shipping_label(order_id)`
- `register_shopmine_shipping_tracking(order_id, carrier, tracking_number)`
- `list_shopmine_shipping_labels(limit=20)`
- `refund_shopmine_shipping_label(shipping_id)`
- `send_order_progress_updates(order_id, to_state, inputs={})`

Order responses expose human-readable UTC time fields ending in `_str`.
Prefer those fields when describing deadlines or order history to a user.
MCP order tools request Unix timestamp fields alongside the `_str` fields for
programmatic use. Order responses include `quantity`, the purchased item count.
Use `status_str` for user-facing status and `status_hint`, when
present, for the next-step explanation.
Physical-good orders include `allows_returns` as agreed at purchase; later
listing changes do not change an existing order's permission. A false value
does not offer carrier-return requests or return-label actions.
Order reads include caller-scoped `seller_next_actions` or
`buyer_next_actions`. Each descriptor names a callable tool, prefilled
arguments, and `required_inputs`; call only a returned action. A paid carrier
sale returns one of `generate_shopmine_shipping_label` or
`register_shopmine_shipping_tracking`, plus the available pre-carrier cancellation
action. The tracking action requires a supported carrier and tracking number.
When `cancellation_penalty` is present, explain that charge to the human before
cancellation. Buyer carrier-return actions use the same descriptor format.
Progress tools determine buyer or
seller from the approved credential. Before carrier
acceptance, the seller can use `send_order_progress_updates(to_state="cancelled")` from `paid`,
`label_generated`, or `label_uploaded`; ShopMine refunds the buyer, restores inventory, and handles any
generated label. Cancellation from `paid` is penalty-free; cancellation from
`label_generated` or `label_uploaded` applies the seller no-ship penalty.
The buyer cannot drive this seller transition. When an order is
`return_label_sent`, buyers ship with the
provided ShopMine label and carrier scans
drive return progress. Do not ask buyers to report `return_tracking`. Prefer
`status_hint` when explaining the current state or next action.
For physical-good item purchases, `paid` means checkout succeeded and inventory
is reserved. For carrier orders, a ShopMine-generated outbound label moves the order to
`label_generated`; a seller-provided label moves it to `label_uploaded`. All
three carrier pre-ship states share the same 48-hour ship-by deadline from `paid`.

Hand-to-hand orders stay `paid` while pickup is arranged. The buyer and seller
read `fulfillment_mode`, `fulfillment_postal_code`, `fulfillment_phone`,
and `fulfillment_buyer_notes` using `get_shopmine_order` or their purchase/sales
history. After receiving the item, the buyer obtains a Delivery PIN in the
order UI or with `get_hand_to_hand_delivery_pin(order_id)` using the buyer's
buy-scoped credential. It returns `delivery_pin`, `expires_in_seconds`, and a
`note` to relay to the buyer: share the PIN only after receiving the item,
because sharing it confirms delivery and allows payment to be charged.
Generating another PIN replaces the previous one. Share it only after receipt.
The seller follows the returned
`send_order_progress_updates` action with `to_state="completed"` and
`inputs={"delivery_pin":"<buyer-provided PIN>"}`. Never ask for the PIN before
the buyer receives the item. Pickup does not use carrier labels, tracking, or
carrier ship-by deadlines. Do not use carrier return or shipping-label tools
for pickup orders; a separate manual pickup return/dispute workflow is not yet
available.


Pickup confirmation optionally accepts `inputs.pickup_evidence` containing
`filename` and `file_base64` for one JPEG, PNG or WebP photo up to 10 MB.
Omit it for PIN-only confirmation. If supplied, the photo must upload successfully
before confirmation. It is seller-supplied pickup evidence, not independently
verified by ShopMine; avoid faces, identity documents and unnecessary personal
information. It is not public or shown to buyers. Seller order reads report
`pickup_evidence_attached=true` when a photo is attached. Action descriptors list
this field in `optional_inputs`, separately from the required `delivery_pin`.

Buyer-paid carrier checkout includes outbound `shippingCents`;
seller-paid shipping is omitted from buyer checkout, and pickup has
`shippingCents=0`. External agents use MCP `buy_shopmine_listing` or the
corresponding A2A tool for checkout. MCP
`generate_shopmine_shipping_label` generates ShopMine outbound labels, while
`register_shopmine_shipping_tracking` registers seller-provided outbound tracking. Seller-paid
ShopMine outbound postage is charged to the seller account without a separate
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
the compact search view. `get_shopmine_listing` returns a prospect, buyer, or owner view
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
`remove_shopmine_listing_media`. The arrays are exclusive, and deletion from either
array deletes that media. MCP accepts base64 files, not media IDs, for add or
replace operations. Only general-media writes run display classification.

## Reauthorize

For `reauthorize` / `replace-keys`:

1. Detect the active client.
2. Check whether `shopmine` is already configured with `codex mcp get shopmine` or `claude mcp get shopmine`.
3. Run the agent auth code flow again with the requested scopes.
4. Replace only the active client's `shopmine` MCP entry after the new code is
   claimed successfully.
5. Tell the user to restart the active client and call a read-only listing tool.

## Uninstall

Codex:

```bash
codex mcp remove shopmine
```

Claude Code local/current project:

```bash
claude mcp remove shopmine
```

Claude Code user/global:

```bash
claude mcp remove shopmine --scope user
```
