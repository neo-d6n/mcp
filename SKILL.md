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
claimed result is a `cli_ke...` OBO credential with the approved combination of
`buy`, `sell`, and/or `payout`. Payout means: Set up payout details, manage or close your payout account, and request payouts of your eligible earnings. The six-digit code is not the final credential and is safe to ask for;
the returned `auth_key` is the credential and must stay secret. Do not ask the
user to paste credentials.

Native MCP clients such as ChatGPT and Claude can use OAuth discovered from
the MCP endpoint. Their human sign-in and consent must remain in the app's
browser flow. A2A also advertises OAuth in its Agent Card and protected-resource
discovery. Use S256 PKCE and the advertised resource for authorization and token
exchange. OAuth access works only on the approved endpoint, lasts 72 hours, and
can be revoked; reconnect to renew it. Do not invent OAuth URLs or automate
human consent. The agent-code flow remains available on both surfaces.

This coding-agent shortcut uses the existing agent-code flow: owner creates a six-digit code at
`/aiauth/create`, agent claims it once at `/aiauth/claim/{code}`, then the
agent uses the returned scoped Bearer credential.

Both MCP and A2A identify the connection as ShopMine Test or ShopMine Production.
Keep credentials and listing/order IDs separate between environments. Invalid
credentials return 401; temporary authorization unavailability returns 503 and
should be retried without replacing credentials. Listing searches preserve
`price_cents_max=0` for free listings; omit a bound or send null to leave it unset.
Listing files require `filename` and exactly one nonempty `upload_id` or
`file_base64`; `content_type` is optional. Prefer the batch upload flow below
for local files; do not read or copy large base64 strings into chat.

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

- `buy` for purchases
- `sell` for listing and sales management
- `payout` to set up payout details, manage or close the payout account, and request payouts of eligible earnings

Request a combination only when the task needs it. Sell never permits buying.
Omitted OAuth scopes still default to buy sell; existing credentials gain no new
permissions automatically. Card purchases require the owner to review and approve
each checkout on ShopMine; saving a card is optional.

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

Tell them to log in, enter the suggested agent name (`<CLIENT_ID>`), select only the requested `buy`, `sell`, and/or `payout` permissions, click Create, and return with the six-digit code. If they already have a code, use it.

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
- Approved result: contains `auth_key`, `client_id`, `scopes`, `expiration_time`. Use `auth_key` immediately. The result is one-shot and the credential expires after 72 hours.

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
- `delete_shopmine_shop(share_id)`: permanently delete an owned empty Shop with
  `sell` scope; returns `{"deleted": share_id}`. A Shop with listings cannot be
  deleted; the request fails without deleting its listings. Available through
  both MCP and A2A.

Listing creation tools require `sell` scope:

- `prepare_shopmine_listing_uploads(files)`: prepare one batch of one to four files. Each manifest entry contains `filename` and positive `size_bytes`. Clients with file and HTTP access upload once before the returned expiry using the exact URL, method, headers, and multipart field name. Keep the returned headers private; a successful transfer returns `files: [{filename, upload_id}]`. Files must match the prepared manifest. Total file bytes must not exceed 100 MB. If preparation expires before uploading, prepare a new batch. Other clients may supply inline base64; local paths and remote file URLs are not inputs.

- `create_physical_good_listing(files, shop_name, title, description, condition, price_usd=None, sale_type=None, auction_starting_bid_cents=None, auction_maturity_seconds=None, inventory_count=None, shipping_enabled=True, local_pickup_enabled=False)`: save core details/media in an explicitly named existing seller-owned Shop. At least one delivery option must be true. Returns real `datum_id`, `state=needs_fulfilment` and computed `next_actions`. Use upload response `files` or inline base64, not `media_ids`; never infer the destination Shop.
- `configure_shopmine_listing_shipping_source(datum_id, source)`: save `source={from_address,item_size}` when shipping is enabled. Address requires name, street, city, region, postal_code and country; package is envelope, small, medium or large.
- `configure_shopmine_listing_pickup(datum_id, pickup)`: save pickup `{postal_code,phone,instructions}` when pickup is enabled. Phone uses international format. Shipping source and pickup details may be offered concurrently; save either first. Each save returns updated next actions or reviewing when all applicable setup is complete.
- `get_shopmine_shipping_quote(datum_id, destination)`: optional when shipping is enabled and source is saved. Supply all six destination fields. Returns `{datum_id,currency,amount_cents,is_estimate:true}` for the all-in label estimate; no label purchase, payment or publication. Final price can change.
- `configure_shopmine_listing_shipping_details(datum_id, config)`: save shipping and return preferences when offered after source and any enabled pickup details are saved. Config contains `outbound` and `inbound` policy choices only; source and pickup details are saved separately and must not be requested again. Pickup-only listings skip shipping setup. Follow returned actions, then read the same listing for review outcome. Use the same tools for later edits; never create again to retry. These tools require sell scope and ownership.

Follow **create → configure → read outcome**. Draft edits and media changes do not publish an unfinished listing. Resume using the ID from owner listings. Never invent an ID, substitute an update for creation, or create another listing after a configuration timeout/conflict: re-read the same ID. Ordinary shipping and visibility updates cannot bypass setup. While reviewing, writes are blocked.

Every fixed-price create-listing call must include `price_usd` as a decimal USD amount of
at least `1.00`, for example `5.43`.
For physical goods, include `inventory_count` when the seller gives on-hand
quantity while creating the listing. It is the total units currently available
for sale; updates set the final count, not an increment.

In `config`, outbound and inbound `mode` identify the payer (`buyer` or `seller`). `outbound.seller_provided_label=false` means ShopMine supplies the label; true means the seller supplies it and requires `seller_flat_shipping_fee` in cents. `inbound.allows_returns` defaults to true; set false only when the seller declines carrier returns. Saved source is shared by outbound shipping and returns. Hand-to-hand returns remain unavailable.
For carrier fulfillment when the outbound payer is the buyer, the item-purchase challenge and response
charge item + platform fee + outbound `shippingCents`. Seller-paid shipping is
omitted from buyer checkout. After requesting an eligible return, the buyer gets
the return label. The return permission, payer and policy agreed at purchase
remain in effect; the buyer or seller pays according to that policy.
MCP/A2A carrier buyers provide `shipping_address` up front or use the OBO owner's
saved profile shipping fallback when available.

Purchases default to carrier when `shipping_enabled=true`, otherwise hand-to-hand
for pickup-only listings. Explicit carrier requires shipping enabled; pickup
requires pickup enabled and needs no shipping address or shipping charge.

Sellers choose carrier shipping and pickup independently with `shipping_enabled` and
`local_pickup_enabled`; at least one must be enabled. These flags may also be
changed through editable listing details. Pickup details belong to the separate
`configure_shopmine_listing_pickup` tool, not shipping preferences. Availability
and area are public; contact details and instructions are shared with the
purchaser only through the paid order.

Only `state=public` listings are discoverable and buyable. Saving a draft or accepting configuration is not publication.

Listing read/manage tools:

- `browse_shopmine_listings(shop_share_id=None, listing_type=None, tags_any=None, languages_any=None, amenities_any=None, price_cents_min=None, price_cents_max=None, currency=None, category=None, subcategory=None, shipping_origin=None, service_type=None, sort="newest", limit=20, cursor=None)`: Shop-scoped feed/discovery view. Use for "what can I buy", "what is available", or "show listings" and do not pass a natural-language query. Every result includes its public Shop reference.
- `search_shopmine_listings(q, shop_share_id=None, listing_type=None, tags_any=None, languages_any=None, amenities_any=None, price_cents_min=None, price_cents_max=None, currency=None, category=None, subcategory=None, shipping_origin=None, service_type=None, sort="relevance", limit=20, cursor=None)`: targeted Shop-scoped search; `q` must be meaningful and non-empty. Every result includes its public Shop reference.
- `search_shopmine_listings_across_shops(q, listing_type=None, tags_any=None, languages_any=None, amenities_any=None, price_cents_min=None, price_cents_max=None, currency=None, category=None, subcategory=None, shipping_origin=None, service_type=None, sort="relevance", limit=20, cursor=None)`: query-only marketplace-wide search for a described item when no Shop context is available or the user asks for options across Shops. If the user has not described what they want, explain that cross-Shop search is available and ask what kind of item they are looking for instead of calling the tool; generic nouns such as listings, items, products, or goods are not a search subject. It does not use or change the current Shop; every result includes a public Shop reference. It requires the configured ShopMine bearer credential.
- `list_my_shopmine_listings(limit=50)`: owner view for listings created by the authenticated user. Every result includes its public Shop reference. Physical-good owner rows include `inventory_count`; physical-good `inventory_count=0` or a missing count means sold out and appears after available listings. Data-listing inventory is not applicable.
- `get_shopmine_listing(datum_id, shop_share_id=None)`: caller-scoped owner view for the seller, buyer view for the purchaser, or prospect view for an authenticated non-purchaser on public listings. The response includes its public Shop reference. The caller-scoped response sets `is_owner=true` only for the seller; never buy that listing.
- `update_shopmine_listing_details(datum_id, fields=None, shop_name=None, price_usd=None, state=None, access_terms=None, seller_notes=None, inventory_count=None, condition=None, brand=None, model=None, color=None, dimensions=None)`: update editable owner fields; requires `sell` scope and ownership. Physical-good `dimensions` is `{x, y, z, weight}`, with x/y/z in inches and weight in ounces. First read the owner view and use only `editable_fields`. `shop_name` moves the listing only to another existing Shop owned by the seller; `price_usd` converts to `price_cents`. Shipping, returns, and pickup are not generic editable fields; use complete fulfillment configuration.
- `get_outbound_shipping_rule(datum_id)`: read a seller-owned listing's outbound shipping and local-pickup settings. Physical-good listing reads do not embed the full rule.
- `get_inbound_shipping_rule(datum_id)`: read the listing's return-shipping payer and `allows_returns` permission. Physical-good listing reads do not embed the full rule.
- `delete_shopmine_listing(datum_id)`: permanently delete a listing owned by the authenticated user; requires `sell` scope and ownership.
- `update_shopmine_listing_media(datum_id, files, replace=False)`: append uploaded references or inline base64 files to a seller-owned listing, or replace the complete media set when `replace=True`; requires `sell` scope and ownership. Listings hold at most four media items, and add rotates out the oldest item when necessary. Physical goods in `needs_fulfilment` remain editable drafts; media updates do not finish setup. Configured listings may enter review; read the listing for the outcome.
- `remove_shopmine_listing_media(datum_id, media_version, media_numbers=None, display_media_numbers=None)`: delete one or more items by visible one-based numbers in the seller's exclusive `media` and/or `display_media` arrays. Deletion from either array deletes that media. Pass the opaque `media_version` from the same read; refresh with `get_shopmine_listing` after a stale-version response. Read the updated listing for its current state and available actions. Requires `sell` scope and ownership.
- `retry_making_listing_public(datum_id)`: request another review of a hidden listing when its owner view permits retry. Follow its correction guidance first, then read it again for the outcome. Requires `sell` scope and ownership.
- `setup_shopmine_payout_account(country="US", display_name=None)`: require `payout` scope and verified email; create or resume US payout setup and return `human_payout_action_required` plus a secure ShopMine URL. Supply only a known display name. The human completes identity, bank details, and agreements.
- `close_shopmine_payout_account(payout_account_ref)`: require `payout` scope, verified email, and an explicit owner request to close the payout account. First read `seller.payout_account_ref` from `get_shopmine_payment_account`; never invent a reference. All balances and pending payouts must be cleared first. Success returns `status="closed"` and the same reference while retaining financial history. Repeating the closed reference is safe and cannot close a replacement account. Closure never cashes out earnings automatically.
- `cashout_shopmine_earnings(currency, request_id)`: require `payout` scope and request all eligible earnings in one currency. Reuse a stable `request_id` for the same cashout. There is no partial-amount input. `processing` or `bank_delivery_pending` does not mean bank receipt; `bank_delivery_action_required` needs the owner's attention. Read the original cashout rather than creating a replacement request. A failed cashout includes a safe message, error_code and retry timing; its reference remains in history. Cashout submission returns 503 for temporary difficulties (MCP includes status_code); account reads still succeed. A failed request stays failed on replay; start a new cashout only when eligible. The returned delay is an eligibility limit, not a promise of success.
- `get_shopmine_payment_account(cursor=None, cashout_request_id=None)`: read masked card/setup readiness and the financial sections allowed by the credential. `buy` permits buyer sections and statements; `payout` permits seller balances, statements, cashout results and `seller.payout_account_ref` (null when no payout account is attached); `sell` alone permits common readiness. The response identifies `included_sections`, returns `statement` and `next_cursor` when permitted. Open setup URLs in the initiating owner's browser; never enter sensitive details or accept agreements for the human, and never treat returning from setup as proof of readiness.
- `get_shopmine_payment_url(datum_id, request_id, quantity=None, bid_amount_cents=None, fulfillment_mode=None, shipping_address=None, booking_start_time=None, booking_end_time=None)`: create or recover one credit-card checkout with `buy` permission. Use a stable request ID of at most 200 characters and unchanged purchase inputs when retrying the same checkout. Returns `status="payment_required"`, a secure `url`, and `expires_at`. The initiating owner opens the URL, signs in, reviews the total and billing/shipping details, and approves payment with an existing saved card or a new card. Saving a new card is optional and off by default. Virtual cards are entered on this browser checkout page like other cards. Card checkout and any available browser wallet or Link option do not issue an SPT; MPP/SPT buying uses its separate credential flow. Never collect card details in chat or approve payment for the human. A URL does not prove payment completed; read the payment account or order for its outcome.
- `buy_shopmine_listing(datum_id, payment_credential=None, quantity=None, request_id=None, bid_amount_cents=None, shipping_address=None, fulfillment_mode=None, booking_start_time=None, booking_end_time=None, params=None)`: purchase with a `buy` credential using MPP/SPT. Call once for the challenge, then retry unchanged purchase inputs with `payment_credential`. For credit-card checkout use `get_shopmine_payment_url` instead. Before purchase, tell the buyer when `allows_returns=false`. Select `fulfillment_mode="hand-to-hand"` only when `local_pickup_enabled=true`; pickup has no shipping charge or address requirement. Carrier delivery requires a deliverable `shipping_address` with name, street, city, region, country and postal_code unless a saved shipping address is available. Invalid addresses or unavailable shipping quotes reject the attempt before payment. Changing fulfillment mode requires a fresh challenge. Successful MPP responses may include `payment_receipt.orderId`; use that order reference for follow-up. An authorized result is not completed payment. For the credential format see `purchase_payments.payment_credential` in `https://shopmine.ai/.well-known/agent.yml`.
- `request_order_return(order_id)`: request a return for a delivered carrier-shipped physical-good purchase with `allows_returns=true`, only when offered in `buyer_next_actions`. It moves the order from `delivered` to `return_requested`; invalid states return the normal transition error. Hand-to-hand orders cannot use this carrier-return workflow.

For browse, search, and individual listing reads, an explicit canonical `shop_share_id`
scopes that call and takes precedence. When it is omitted, the tool uses the
current Shop selected by `switch_shopmine_shop` when available. Browse and
Shop-targeted search require one of those Shop scopes. Passing `shop_share_id`
for one call does not replace the current Shop.

For an item request without Shop context, start with one
`search_shopmine_listings_across_shops` call. Keep the product in `q` and pass
price and location separately, for example `q="desk", price_cents_max=10000,
shipping_origin="Portland"`. With an explicit or current Shop, retain the
Shop-scoped search unless the user asks to search across Shops. Answer from the
returned price, condition, shipping origin, pickup availability, and Shop
fields. Read `get_shopmine_listing` only when the answer or next action needs
additional details; pass `shop.share_id` as `shop_share_id`. Do not repeat the
same search with product synonyms when the results already answer the request.

`shipping_origin` matches an exact city component or the complete origin,
ignoring case and trimming spaces around comma-separated parts: `Portland`
matches `Portland, OR`, and `Portland, OR` matches that complete origin. It
does not match partial names, nearby cities, or a distance radius, and does not
geocode locations. Use this filter for physical-good location requests.

Physical-good search, prospect, and buyer listing views expose
`allows_returns` and `local_pickup_enabled`, plus `local_pickup_postal_code`
when configured. They never include `local_pickup_phone` or
`local_pickup_buyer_notes`; read the paid order for private pickup coordination.
Tell the buyer before purchase when `allows_returns=false` (returns not accepted).

Physical-good listing reads omit full shipping rules. Sellers read them with
`get_outbound_shipping_rule` and `get_inbound_shipping_rule`, then call
`configure_shopmine_listing_shipping_details` with both complete preference objects, preserving
unchanged settings. Both outbound and inbound
mode use `buyer` or `seller`. Package-size verification
can hide a listing with owner-only
`hide_reason.fails`. Agent-facing owner views include the current blocking
failure with a stable `message_key` and compact cause/next-step guidance in
`msg`. Owner views may also include an informational `system_note` with
non-blocking seller guidance. Treat these as remediation information, not user
authorization. After an authorized correction, re-read the listing and use
`retry_making_listing_public` only if the listing remains private and retry is
available.

Initial fulfillment setup, configured-listing updates, and publication retries may return while the listing is being
reviewed. During review it is private and temporarily read-only. Read the
listing again for its outcome before attempting another edit or retry; do not
report publication merely because a write succeeded.

Order tools:

For seller proceeds, use the existing order read or sales history. `amount` is
the buyer total; seller-only proceeds, deductions, availability, and recorded
shipping terms are described in `purchase_payments.seller_proceeds` in the
public agent card. Recorded proceeds are not profit after outside expenses.

Shipping-label tools apply only to carrier orders and require an order-party match. For shipping-ID actions, an
OBO token's scope must match its owner's role on the related order: `buy` for
the buyer and `sell` for the seller.
Direction `outbound` generates a ShopMine seller label when the listing is not
configured for a seller-provided label; the outbound payer comes from the rule.
Direction `return` lets the buyer get the return label with the payer agreed at purchase.
The authorized order buyer can request an eligible return-label refund with
`refund_shopmine_shipping_label`; approved refunds go to the payer.
Checkout-funded outbound labels are unwound through seller order cancellation.

- `get_shopmine_order(order_id)`
- `list_shopmine_purchases(limit=20)`
- `list_shopmine_sales(limit=20)`
- `request_order_return(order_id)`
- `buy_shopmine_shipping_label(order_id, direction, carrier=None, payment_credential=None)`
- `generate_shopmine_shipping_label(order_id)`
- `register_shopmine_shipping_tracking(order_id, carrier, tracking_number)`
- `list_shopmine_shipping_labels(limit=20)`
- `refund_shopmine_shipping_label(shipping_id)`
- `send_order_progress_updates(order_id, to_state, inputs={})`

For carrier orders, the buyer is charged when outbound shipping is confirmed. Ship only after the order shows label_generated or label_uploaded. If confirmation is pending, retry the same returned shipping action and wait; do not buy another label or ship yet.

Use `list_shopmine_shipping_labels` to retrieve existing labels and the latest
recorded carrier tracking status for outbound and return shipments. Carrier
tracking status is separate from order status.

Order responses expose human-readable UTC time fields ending in `_str`.
Prefer those fields when describing deadlines or order history to a user.
MCP order tools request Unix timestamp fields alongside the `_str` fields for
programmatic use. Order responses include `quantity`, the purchased item count.
Seller order reads and sales history include the recipient name and postal
address when needed for shipping, without the buyer's phone number or email address.
Use `status_str` for user-facing status and `status_hint`, when
present, for the next-step explanation.
Carrier returns must be requested and handed to the carrier within the original
fourteen days after delivery. A request, label or retry does not pause, reset or
extend that deadline. A request or unused label alone does not trigger a refund;
follow the order deadline and returned actions.
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
For fixed-price physical-good purchases, or winning auction purchases, `paid` means the purchase is confirmed and inventory
is reserved. For carrier orders, a ShopMine-generated outbound label moves the order to
`label_generated`; a seller-provided label moves it to `label_uploaded`. All
three carrier pre-ship states share the same 48-hour ship-by deadline from `paid`.

Hand-to-hand pickup must be confirmed within three days after successful
checkout, or the transaction is cancelled without a no-ship penalty. Paid
order reads include the pickup `cancellation_deadline` (or its formatted
`cancellation_deadline_str`); Delivery PIN issuance and confirmation after that
deadline are rejected. Do not hand over the item after the deadline.
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
`register_shopmine_shipping_tracking` registers seller-provided outbound tracking. Follow the order’s configured payer and displayed shipping price. Direction `return` lets the buyer get the return label after an eligible return
request, with the payer agreed at purchase. Use payment credentials for item
checkout or buyer-paid return labels only when an actual payment challenge is
returned. Seller-paid return labels do not require buyer payment.

Use `datum_id` as the listing identifier. For create tools, `files` is required
and contains one to four entries such as
`{"filename":"photo.jpg","upload_id":"..."}` from the binary upload response,
or `{"filename":"example.pdf","file_base64":"..."}` for inline uploads.
Never send both source keys for one file, even if one is empty or null. Do not use the old `session`
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
array deletes that media. MCP accepts uploaded file references or inline base64
files for add or replace operations; `media_ids` is not a public input. Read the updated listing for its current state and available actions.

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

## Auction sale type

Physical goods choose `sale_type="fixed_price"` (default) or `"auction"` independently of fulfillment. Fixed-price creation requires `price_usd`; auction creation requires `auction_starting_bid_cents` (at least 100 USD cents). The optional `auction_maturity_seconds` supports 1–129600 seconds and defaults to one day. These are editable seller listing fields. Auction listing reads expose `auction_current_bid_cents`, `auction_next_bid_cents`, `auction_minimum_increment_cents`, `auction_maturity_seconds`, `auction_matures_at` and `auction_round_state` along with the starting bid.

For MPP auctions reuse `buy_shopmine_listing` with `bid_amount_cents` and a stable `request_id` of at most 200 characters. Keep the same ID, bid, fulfillment and address when retrying payment proof. Card bidding uses `get_shopmine_payment_url` with `bid_amount_cents`; the human approves the binding bid on ShopMine. The listing chooses its sale type; clients cannot set order state. A winning bid is binding. Each accepted higher bid restarts the countdown. Check the owned order for `bid_placed` (leading), `outbid`, `bid_rejected`, or `paid` (winner), and use its bid maturity timestamp. Authorization alone does not prove leadership. A `processing` result means retry the same intent or read its original order; do not create another bid to resolve uncertainty. A pending hold release is not confirmation that funds are available.

Bidding starts no shipping or pickup deadline. The winning purchase starts its displayed fulfillment deadline and keeps its accepted fulfillment and return terms. Sellers may withdraw an auction before a win; afterward the ordinary pre-fulfillment cancellation and return rules apply. Remaining or newly available stock can be offered in a fresh auction while earlier winners fulfill. Old bids never carry into that auction. See the [Terms](https://shopmine.ai/home/terms/) and [Privacy Policy](https://shopmine.ai/home/privacy/) for obligations and bid-data visibility.

Refunds may be processing or issued. Follow the order’s current refund guidance. An issued refund goes to the original payment method; the bank may take 5–10 business days to show it. If the order says the refund needs attention, contact ShopMine support. A released payment hold is separate from a refund.
