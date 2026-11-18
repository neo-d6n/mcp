# ShopMine MCP OAuth Setup

This repository contains the optional `shopmine` skill for connecting or
reauthorizing the active AI client through ShopMine OAuth. No skill installation
is required. The public connection guide is https://shopmine.ai/llms.txt.

## Connect

Add https://mcp.shopmine.ai/mcp through your AI client's supported remote MCP or
connector settings. Browse public Shops and listings as a guest where your client supports it; use OAuth for account actions or clients requiring sign-in. The human signs in to ShopMine and
explicitly approves only the permissions needed for the task: `buy` for purchases
and buyer orders, `sell` for Shops, listings and seller fulfillment, `payout` for
payout setup, account management or closure and eligible earnings payouts, and
`support` for seller messaging. Omitted scopes default to `buy sell`; existing
grants gain no permissions automatically. Sell does not authorize buying.

AI readers should give 3–5 short numbered manual setup steps for the person's
current app and platform, using verified official menu and button names. Infer
the app from context; ask one concise question only if unknown. Include the
exact server URL, the guest browsing or OAuth option, and the app's final
install, enable or select-in-chat step. ChatGPT web uses a plugin; Claude and Grok
use connectors; ChatGPT desktop and editors use MCP-server settings. The connection guide links
current official instructions. If you cannot add the server from chat, say so
briefly and still give those manual steps. Use a documented web or desktop path
when mobile setup controls are unavailable. A prompt or fetched guide alone
does not connect the app. Verify guest browsing with a public Shop search, or use `profile_info()` after OAuth to confirm the account and scopes.

MCP and A2A support guest public browsing. Account actions require OAuth. A2A guest clients must retain and resend the server's guest cookie; a conversation ID alone is insufficient. Clients without cookie support should use OAuth. Guest A2A usage is limited to four messages per minute, and guest conversations expire three days after creation. After sign-in, start a new authenticated context or explicitly resume an authorized existing one; guest history does not transfer. If the guest cookie expires, discard it and send a new message without task/context IDs.
Both identify ShopMine Test or ShopMine Production; keep credentials and
listing/order IDs separate.

Use the original OAuth connection to resume an A2A conversation or task. Token refresh preserves access. A different app or fresh consent starts a new history.

Clients discover OAuth metadata, use S256 PKCE and the advertised resource for
authorization and token exchange, and send access tokens only to the approved
resource. Native OAuth clients manage token storage and refresh. Keep tokens
private; never paste credentials into chat or automate human login or consent.
Account owners can review and revoke access in ShopMine account settings.

After authorization, initialize MCP and use its runtime discovery for the
current tool descriptions and schemas. Verify the intended account, environment
and approved `token_scope` with `profile_info()`. A saved connection alone does
not prove readiness. For A2A, use an A2A-capable client and the current Agent Card
at https://a2a.shopmine.ai/.well-known/agent-card.json.

Every credit-card checkout requires the human to review and approve payment on
ShopMine. Saving a card is optional and does not approve future purchases.
Never collect card, bank or identity-verification details in chat.

## Optional skill installation

After cloning this repository, install the skill from its root:

```bash
npx skills add .
```

For global installation, use `npx skills add . --global`. Then run `/shopmine`
in your AI coding agent. Run `/shopmine reauthorize` to renew consent or request
additional permissions through the same OAuth flow.

[SKILL.md](SKILL.md) describes secure refresh, reauthorization and verification.
The live MCP server is the authority for its tool catalog and schemas; the A2A
Agent Card is the authority for its supported interface and skills. This optional
setup helper does not duplicate those catalogs.
