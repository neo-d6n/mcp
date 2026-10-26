# ShopMine MCP Install Skill

This repository contains the optional `shopmine` skill for installing and
configuring the ShopMine MCP tools in the active AI client.

ShopMine does not require a skill. Agents can discover the public agent surface at
`https://shopmine.ai/.well-known/agent.yml` or `https://shopmine.ai/llms.txt`, claim a
human-created ShopMine agent auth code, and configure `https://mcp.shopmine.ai/mcp` directly.
This skill is only a convenience shortcut for clients that support skills.

ChatGPT and Claude can also connect through native MCP OAuth: add the MCP server
URL in the app, then sign in to ShopMine and approve the needed buy, sell, and/or payout access.
OAuth access expires after 72 hours and supports client revocation. A2A also
offers OAuth through its Agent Card and protected-resource discovery. Use S256
PKCE and the advertised resource for authorization and token exchange; each
token works only on the endpoint approved by the human. Reconnect to renew it.
This skill's direct Bearer installation and both surfaces' agent-code flow
remain available.

Both MCP and A2A identify the connection as ShopMine Test or ShopMine Production.
Keep credentials and listing/order IDs separate between environments. Invalid
credentials return 401; temporary authorization unavailability returns 503 and
should be retried without replacing credentials. Listing searches preserve
`price_cents_max=0` for free listings; omit a bound or send null to leave it unset.
Listing files require `filename` and exactly one nonempty `upload_id` or
`file_base64`; `content_type` is optional. For local files, prefer
`prepare_shopmine_listing_uploads` when the client can read files and send HTTP
requests. It returns a short-lived upload URL, method, headers, expiry, and
`field_name`. Send the originals together in one multipart request, then pass
the returned `files` to create or update. Keep the returned upload headers private.
Do not read or copy large base64 strings into chat. Clients without file/HTTP
access may use inline `file_base64` when available; there is no local-path or
remote-file-URL input.

## Install

After cloning this repository, install the skill from its root with the Skills
CLI:

```bash
npx skills add .
```

To install globally instead of only for the current project:

```bash
npx skills add . --global
```

## Use

After installing, open your AI coding agent and run:

```text
/shopmine
```

The skill asks which buy, sell, and/or payout permissions your agent needs, asks you
to create a six-digit code at `https://shopmine.ai/aiauth/create`, claims that code,
then stores the returned 72-hour scoped credential in your MCP config:

```text
https://mcp.shopmine.ai/mcp
```

The public agent contract in `https://shopmine.ai/.well-known/agent.yml` and
`https://shopmine.ai/llms.txt` is the source of truth. `SKILL.md` implements the
same human approval flow as an optional shortcut. After setup, the current MCP
surface supports Shop selection/management, listing search/create/manage, buyer purchase history, seller
sales history, carrier-order returns, outbound-label generation, return-label
purchase/refund, and seller fulfillment by shipping or local pickup.

Physical-good creation is standardized as **create → configure → read outcome**.
Use `prepare_shopmine_listing_uploads(files)` for one to four local files, supplying
`filename` and positive `size_bytes` (100 MB total maximum).
Send one binary multipart batch before its expiry. Call
`create_physical_good_listing` with core details and the upload response’s `files`.
Existing inline base64 files remain supported. Each file must have exactly one
source key; both keys are invalid even if one is empty or null.
Choose `shipping_enabled` (default true) and `local_pickup_enabled` (default false)
on create; at least one must be enabled. The response gives a real `datum_id`,
`state=needs_fulfilment` and computed `next_actions`. Follow those actions on the
same ID. When both delivery options are enabled, shipping source and pickup
setup are available concurrently; save either first.
`configure_shopmine_listing_shipping_source(datum_id, source)` saves
`from_address` (name, street, city, region, postal_code, country) and `item_size`
(envelope, small, medium, large). `configure_shopmine_listing_pickup(datum_id,
pickup)` saves `postal_code`, international-format `phone`, and `instructions`.
Shipping preferences become available after source and any enabled pickup details
are saved; pickup-only listings need no shipping setup. Save shipping/return
policy choices through `configure_shopmine_listing_shipping_details(datum_id, config)`;
its outbound/inbound objects contain neither source nor pickup details.
Saves return updated actions or `reviewing` when applicable setup is complete.
With shipping enabled and source saved, optional `get_shopmine_shipping_quote(datum_id,
destination)` accepts a complete six-field destination and returns
`{datum_id,currency,amount_cents,is_estimate:true}` for the all-in label estimate.
No label purchase, payment or publication occurs; final price can change.
These tools require sell scope and ownership; reuse saved details without asking again.
Read the same listing for reviewing → public or private with correction guidance.
Drafts remain editable and resumable. Only public listings are discoverable and
buyable. On conflict or a lost response, re-read that ID; never create another.
Ordinary shipping or visibility updates cannot bypass setup.
ShopMine validates listing-type fields on create and update calls. Callers receive
the declared response or validation/auth error. Seller-paid shipping does not
appear in buyer checkout. Physical-good create calls may include `inventory_count` when
the seller gives on-hand quantity. It is the total units currently available
for sale; updates set the final count, not an increment. Owner listing lists include physical-good
`inventory_count`; physical-good `inventory_count=0` or a missing count means
sold out and appears after available listings. Data-listing inventory is not
applicable. See `SKILL.md` and `llms.txt` for the full
create/update/shipping field contract. Listing updates use
`update_shopmine_listing_details` and the owner view's `editable_fields` list;
`shop_name` moves an owned listing to another existing Shop owned by that seller;
physical-good listing reads omit full shipping rules. Sellers read them with
`get_outbound_shipping_rule` and `get_inbound_shipping_rule`, then use
`configure_shopmine_listing_shipping_details` with both complete preference objects.
Each rule's `mode` selects the payer (`buyer` or `seller`).
`seller_provided_label=false` means ShopMine supplies the label; true means seller
supplies it and requires `seller_flat_shipping_fee` in cents. Saved package and
address are shared by shipping and returns.
Return permission is `config.inbound.allows_returns`. Preserve unchanged
preferences. Pickup details are saved separately; delivery-option flags may be
changed through editable listing details. Detailed fulfillment fields remain
excluded from generic updates and `editable_fields`.
Listing discovery and detail reads expose `allows_returns`. Tell the buyer
before purchase when it is false (returns not accepted). If package-size or
outbound-shipping verification hides a physical-good listing,
agent-facing owner views include the current blocking failure with a stable
`message_key` and compact cause/next-step guidance in `msg`. Treat that guidance
as remediation information, not user authorization. Owner views may also
include an informational `system_note` with non-blocking seller guidance. After
an authorized correction, re-read the listing and call
`retry_making_listing_public` only if it remains private and retry is available.
Initial setup, configured-listing updates, and publication retries may return while a listing is being
reviewed. During review it is private and temporarily read-only; read it again
for the outcome. A successful write does not itself mean the listing is public.
Every listing create explicitly names an existing seller-owned Shop; current
browse state is never used as the listing destination. MCP exposes
`switch_shopmine_shop`, `get_current_session_shopmine_shop`, `create_shopmine_shop`,
`list_my_shopmine_shops`, `browse_shopmine_shops`, `search_shopmine_shops`, `get_shopmine_shop`, and
`update_shopmine_shop`, and `delete_shopmine_shop`. Shop search accepts an optional query: nonblank text returns
the server-ranked matching list, while omitted or blank text returns
recommended public ShopMine Shops. Shop browse also returns recommended public ShopMine
Shops and their public Shop identity. Excessive Shop search calls may be
throttled. The current-session
tool returns the canonical `shop_share_id`, or `null` when no Shop is selected;
a temporary session-service failure is returned as an error instead. Each owner
may create at most four Shops. `delete_shopmine_shop(share_id)` requires `sell`
permission and ownership, permanently deletes an empty Shop, and returns
`{"deleted": share_id}`. A Shop with listings cannot be deleted; the request
fails without deleting its listings. Both MCP and A2A support this operation.
Seller `get_shopmine_listing` reads expose media through one-based `media` and
`display_media` descriptors plus an opaque `media_version`, without raw media
IDs. Add or replace up to four uploaded references or inline base64 files with
`update_shopmine_listing_media`; appending beyond four rotates out the oldest item.
Remove media by passing visible numbers from either `media` or `display_media`
and the matching version to `remove_shopmine_listing_media`. The arrays are
exclusive. Deletion from either array deletes that media; only general-media
writes run display classification.
`browse_shopmine_listings`, `search_shopmine_listings`, and `get_shopmine_listing` accept an
optional canonical `shop_share_id`. An explicit value scopes that call and
takes precedence; when omitted, the tools use the current Shop selected by
`switch_shopmine_shop` when available. Browse and Shop-targeted search require one
of those Shop scopes. Explicit per-call selection does not replace the current
Shop. Browse returns compact search-view listings for feed/discovery
requests such as "what can I buy" or "show listings". Search returns the same
Shop-scoped view for targeted item searches and requires a meaningful non-empty
`q`. Individual reads return the caller-scoped owner, buyer, or prospect view
with `is_owner=true` only for the owner; physical-good full reads may
include a curated display-media count. Every browse, search, individual-read,
and owned-listing result includes its public Shop name, share ID, and URL.
Physical-good search, prospect, and buyer listing views include
`local_pickup_enabled` and, when configured, `local_pickup_postal_code`.
Private `local_pickup_phone` and `local_pickup_buyer_notes` are never part of
those listing views; pickup coordination is available on the paid order.
`search_shopmine_listings_across_shops` is the query-only marketplace-wide search.
It does not accept or inherit a Shop or change the current Shop. It requires the
same approved ShopMine bearer credential as every MCP operation.
If the user has not described an item, tell them that cross-Shop search is
available and ask what kind of item they want instead of calling it with a
generic noun such as `listings`, `items`, `products`, or `goods`.

For a described item without Shop context, start with one cross-Shop search;
retain Shop-scoped search when a Shop is explicit or current. Keep the product
in `q` and use separate filters, for example
`search_shopmine_listings_across_shops(q="desk", price_cents_max=10000,
shipping_origin="Portland")`. `shipping_origin` matches the exact city
component or complete origin, ignoring case and trimming spaces around
comma-separated parts: `Portland` matches `Portland, OR`. It is not substring,
radius, or geocoded search. Answer from the returned price, condition, origin,
pickup, and Shop fields; use `get_shopmine_listing` with `shop.share_id` as
`shop_share_id` only for needed additional details. Avoid repeated synonym
searches when results already answer the request.

For credit-card purchases, call `get_shopmine_payment_url` with `buy` permission,
`datum_id`, and a stable `request_id` (at most 200 characters). Optional purchase
inputs are `quantity`, `fulfillment_mode`, `shipping_address`, `booking_start_time`
and `booking_end_time`. Reuse the same ID and unchanged inputs for the same
checkout. The response contains `status="payment_required"`, `url`, and
`expires_at`. The initiating owner opens the URL, signs in, reviews the total
and billing/shipping details, and approves payment using an existing saved card
or a new card. Saving a new card is optional and off by default. Never collect
card details in chat or approve payment for the human. A URL is not proof of
payment; read the payment account or order for its current outcome.

`buy_shopmine_listing` retains the x402/MPP challenge and credential flow for
MPP/SPT purchases. Never switch payment methods silently. A virtual card is entered
on the secure browser checkout page like another card. Card checkout and any
available browser wallet or Link option do not issue an SPT; obtain an SPT
through the separate MPP/SPT flow when using `buy_shopmine_listing`.

Both MCP and A2A also expose exactly these account operations:

- `setup_shopmine_payout_account(country="US", display_name=None)` requires `payout` scope and verified email, supports US accounts, and returns `human_payout_action_required` with a secure setup URL. Supply only known plain display text; the human provides identity and bank details and accepts agreements.
- `close_shopmine_payout_account(payout_account_ref)` requires `payout` scope, verified email, and an explicit owner request to close the payout account. Read the current opaque reference from `seller.payout_account_ref` in `get_shopmine_payment_account`. All balances and pending payouts must be cleared first. Success returns `status="closed"` and the same reference; financial history is retained. Repeating that closed reference is safe and cannot close a replacement account. Closure never cashes out earnings automatically.
- `cashout_shopmine_earnings(currency, request_id)` requires `payout`, selects all eligible earnings in one currency, and reuses the stable ID for retries. There is no partial-amount input. A pending result is not bank receipt; follow `bank_delivery_action_required` when returned and check the original cashout. A failed cashout includes a safe message, error_code and retry timing; its reference remains in history. Cashout submission returns 503 for temporary difficulties (MCP includes status_code); account reads still succeed. A failed request stays failed on replay; start a new cashout only when eligible. The returned delay is an eligibility limit, not a promise of success.
- `get_shopmine_payment_account(cursor=None, cashout_request_id=None)` returns masked setup readiness and authorized financial sections with `included_sections`. Buy permits buyer sections and statements; payout permits seller balances, statements, cashout results and `seller.payout_account_ref` (null when no payout account is attached); sell alone permits common readiness. Use `next_cursor` for the next statement page.

Payout permission means: Set up payout details, manage or close your payout
account, and request payouts of your eligible earnings. Sell and payout do not authorize a purchase. Open returned
setup links in the initiating owner's signed-in browser and complete the action
there; do not collect sensitive details or accept agreements in chat. Returning
from a setup link does not establish readiness. Omitted OAuth scopes continue to
default to buy sell.

Build `payment_credential` by keeping the returned challenge fields at the JSON
top level and adding the selected method and issued SPT, then encode that JSON
as `MPP <base64url(JSON)>`:

```javascript
const credential = {
  ...challenge,
  method: { scheme: "stripe", type: "spt" },
  token: issuedSpt
};
```

Keep `challengeId`, `created`, and `signature` unchanged and at the top level;
do not nest the challenge. Keep the credential private and retry the same
purchase arguments before the challenge expires.
The typed `fulfillment_mode` purchase input accepts `carrier` or `hand-to-hand`;
carrier is the physical-good default. Choose hand-to-hand only when the listing
returns `local_pickup_enabled=true`. Pickup requires no shipping address; its
challenge identifies `fulfillmentMode="hand-to-hand"` and includes `itemCents`,
`platformFeeCents`, and `shippingCents=0`. Repeat the selected `fulfillment_mode`
on the payment-credential retry. Changing mode requires a fresh challenge.
Successful checkout includes `payment_receipt.orderId`; pass that value as
`order_id` to `get_shopmine_order` and subsequent order actions.
Seller order reads and sales history include the recipient name and postal
address when needed for shipping, without the buyer's phone number or email address.
Seller order reads and sales rows keep `amount` as the buyer total. They can
also return `seller_platform_fee_cents`, `seller_shipping_charge_cents` after
shipping credits, and `seller_net_cents` for recorded proceeds.
`seller_proceeds` reports this order's currency and pending, held, available,
payout-pending, and paid-out amounts; it is not an account-wide cashout balance.
Recorded proceeds exclude costs the seller pays outside ShopMine, such as
self-purchased postage. `outbound_shipping_mode` identifies the buyer or seller
as payer, and `seller_provided_label` distinguishes seller-provided from
ShopMine-generated labels, using the purchase terms when recorded. Unavailable
values are omitted, not estimated. These fields are not returned to buyers.
For carrier orders, the buyer is charged when outbound shipping is confirmed. Ship only after the order shows label_generated or label_uploaded. If confirmation is pending, retry the same returned shipping action and wait; do not buy another label or ship yet.

Carrier purchases require a
ship-to address with `name`, `street`, `city`, `region`, `country`, and
`postal_code`, unless ShopMine can use the OBO owner's saved profile shipping address
as a read-only fallback. ShopMine validates ship-to and ship-from before rating. An
invalid address or unavailable live quote rejects the purchase before any
payment challenge or charge. The
x402/MPP challenge includes the total amount plus
`itemCents`, `platformFeeCents`, and buyer-paid outbound `shippingCents`.
Buyer-paid ShopMine outbound postage is included in carrier checkout.
Follow the order’s configured payer and displayed shipping price. Order responses expose
caller-scoped action descriptors in `seller_next_actions` or
`buyer_next_actions`; each names a callable tool, prefilled arguments, and any
`required_inputs`. Paid carrier sales expose one of
`generate_shopmine_shipping_label` or `register_shopmine_shipping_tracking`, plus the
available pre-carrier cancellation action. When `cancellation_penalty` is
present, explain that charge to the human before cancellation. Return labels use
`buy_shopmine_shipping_label`; label history and refunds use
`list_shopmine_shipping_labels` and
`refund_shopmine_shipping_label`. Label history also provides the latest recorded
carrier tracking status for outbound and return shipments, separately from order
status. After a return request, the buyer gets the return label through the same
action. The return permission, payer and policy agreed at purchase remain in
effect for that order. Buyer-paid labels return a payment challenge; seller-paid
labels do not require buyer payment. Return-label prices expose shipping label,
platform fee and total.
The authorized order buyer can request eligible return-label refunds; approved refunds go to the payer.
Checkout-funded outbound labels are handled through seller order cancellation.
Shipping-ID actions require the OBO scope to match the credential owner's role
on the related order: `buy` for the buyer and `sell` for the seller.
Buyer purchase history uses `list_shopmine_purchases`; seller sales history uses
`list_shopmine_sales`. Delivered carrier-shipped physical-good purchases with
`allows_returns=true` can request a return, when offered in `buyer_next_actions`, with
`request_order_return`, which moves the order to `return_requested`; the buyer
then gets a return label with `buy_shopmine_shipping_label(direction="return")`.
The policy agreed at purchase determines whether the buyer or seller pays.
Invalid states return the normal transition error. Order responses
include `status_str` for user-facing status labels such as `Return Requested`
and `Cancelled`, and may include `status_hint` for user-facing next steps.
Carrier returns must be requested and handed to the carrier within the original
fourteen days after delivery. A request, label or retry does not pause, reset or
extend that deadline. A request or unused label alone does not trigger a refund;
follow the order deadline and returned actions.
Physical-good orders retain their purchase-time `allows_returns` value; changing
the listing later does not change existing orders. False removes carrier-return
requests and return-label actions. Hand-to-hand returns remain unavailable
regardless of that flag.

Account/profile lookups use `profile_info()`. It returns minimal account data
for the authenticated caller: `username`, `email_verified`, and `token_scope`
when the bearer credential is an OBO token.
Progress tools determine buyer or seller from the approved credential. Before
carrier acceptance, the seller can use `send_order_progress_updates(to_state="cancelled")` from `paid`,
`label_generated`, or `label_uploaded`. ShopMine refunds the buyer, restores inventory, and handles any
generated label. Cancellation from `paid` is penalty-free; cancellation from
`label_generated` or `label_uploaded` applies the seller no-ship penalty.
The buyer cannot drive this seller transition. When an
order is `return_label_sent`, buyers ship with the
provided ShopMine return label and carrier scans drive return progress. Do not ask
buyers to report `return_tracking`. Prefer `status_hint` when explaining the
current state or next action.
For fixed-price physical-good purchases, or winning auction purchases, `paid` means the purchase is confirmed and inventory
is reserved. For carrier orders, a ShopMine-generated outbound label moves the order to
`label_generated`; a seller-provided label moves it to `label_uploaded`. All
three carrier pre-ship states share the same 48-hour ship-by deadline from `paid`.

Hand-to-hand pickup must be confirmed within three days after successful
checkout, or the transaction is cancelled without a no-ship penalty. Paid
order reads include the pickup `cancellation_deadline` (or its formatted
`cancellation_deadline_str`); Delivery PIN issuance and confirmation after that
deadline are rejected. Do not hand over the item after the deadline.
Hand-to-hand orders stay `paid` while pickup is arranged. Buyer and seller order
reads include `fulfillment_mode`, `fulfillment_postal_code`,
`fulfillment_phone`, and `fulfillment_buyer_notes`; unpaid orders do not
disclose pickup notes. After receiving the item, the buyer obtains a Delivery
PIN in the order UI or through `get_hand_to_hand_delivery_pin(order_id)` using
the buyer's buy-scoped credential. It returns `delivery_pin`,
`expires_in_seconds`, and a `note` to relay to the buyer: share the PIN only
after receiving the item, because sharing it confirms delivery and allows
payment to be charged. A new PIN replaces the previous one. Share it with the
seller only after receipt. The seller follows the returned
`send_order_progress_updates` action with `to_state="completed"` and
`inputs={"delivery_pin":"<buyer-provided PIN>"}`. Never ask for the PIN before
the buyer receives the item. Pickup orders do not use carrier labels, tracking,
or carrier ship-by deadlines. They cannot use carrier return or shipping-label
tools; a separate manual pickup return/dispute workflow is not yet available.


Pickup confirmation optionally accepts `inputs.pickup_evidence` containing
`filename` and `file_base64` for one JPEG, PNG or WebP photo up to 10 MB.
Omit it for PIN-only confirmation. If supplied, the photo must upload successfully
before confirmation. It is seller-supplied pickup evidence, not independently
verified by ShopMine; avoid faces, identity documents and unnecessary personal
information. It is not public or shown to buyers. Seller order reads report
`pickup_evidence_attached=true` when a photo is attached. Action descriptors list
this field in `optional_inputs`, separately from the required `delivery_pin`.

The skill detects whether it is running under Codex or Claude Code and writes to the matching MCP config:

- Codex: `~/.codex/config.toml`, with the bearer token referenced through `SHOPMINE_MCP_BEARER`
- Claude Code: `~/.claude.json` / Claude MCP scopes, using `claude mcp add`

To reauthorize later:

```text
/shopmine replace-keys
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

## Auction sale type

Physical goods choose `sale_type="fixed_price"` (default) or `"auction"` independently of fulfillment. Fixed-price creation requires `price_usd`; auction creation requires `auction_starting_bid_cents` (at least 100 USD cents). The optional `auction_maturity_seconds` supports 1–129600 seconds and defaults to one day. These are editable seller listing fields. Auction listing reads expose `auction_current_bid_cents`, `auction_next_bid_cents`, `auction_minimum_increment_cents`, `auction_maturity_seconds`, `auction_matures_at` and `auction_round_state` along with the starting bid.

For MPP auctions reuse `buy_shopmine_listing` with `bid_amount_cents` and a stable `request_id` of at most 200 characters. Keep the same ID, bid, fulfillment and address when retrying payment proof. Card bidding uses `get_shopmine_payment_url` with `bid_amount_cents`; the human approves the binding bid on ShopMine. The listing chooses its sale type; clients cannot set order state. A winning bid is binding. Each accepted higher bid restarts the countdown. Check the owned order for `bid_placed` (leading), `outbid`, `bid_rejected`, or `paid` (winner), and use its bid maturity timestamp. Authorization alone does not prove leadership. A `processing` result means retry the same intent or read its original order; do not create another bid to resolve uncertainty. A pending hold release is not confirmation that funds are available.

Bidding starts no shipping or pickup deadline. The winning purchase starts its displayed fulfillment deadline and keeps its accepted fulfillment and return terms. Sellers may withdraw an auction before a win; afterward the ordinary pre-fulfillment cancellation and return rules apply. Remaining or newly available stock can be offered in a fresh auction while earlier winners fulfill. Old bids never carry into that auction. See the [Terms](https://shopmine.ai/home/terms/) and [Privacy Policy](https://shopmine.ai/home/privacy/) for obligations and bid-data visibility.

Refunds may be processing or issued. Follow the order’s current refund guidance. An issued refund goes to the original payment method; the bank may take 5–10 business days to show it. If the order says the refund needs attention, contact ShopMine support. A released payment hold is separate from a refund.
