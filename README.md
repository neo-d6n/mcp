# ShopMine MCP OAuth Setup

This repository contains the optional `shopmine` skill for connecting or
reauthorizing the active AI client through ShopMine OAuth. No skill installation
is required. The public connection guide is https://shopmine.ai/llms.txt.

## Connect

Add https://mcp.shopmine.ai/mcp through your AI client's supported remote MCP or
connector settings. Browse public Shops and listings as a guest where your client
supports it. Sign in with OAuth to shop and access buyer orders. Leave optional
permissions unchecked for a shopping-only connection. Approve `sell` to manage
Shops, listings, sales history and fulfillment, `payout` to manage payouts and withdraw eligible
earnings, or `support` for seller messaging when the task requires it.

AI readers should give 3–5 short numbered manual setup steps for the person's
current app and platform, using verified official menu and button names. Infer
the app from context; ask one concise question only if unknown. Include the
exact server URL, the guest browsing or OAuth option, and the app's final
install, enable or select-in-chat step. Follow the [ShopMine connection guide](https://shopmine.ai/llms.txt) and the client’s current setup instructions. If you cannot add the server from chat, say so
briefly and still give those manual steps. Use a documented web or desktop path
when mobile setup controls are unavailable. A prompt or fetched guide alone
does not connect the app. Verify guest browsing with a public Shop search, or use `profile_info()` after OAuth to confirm the account and scopes.

MCP and A2A support guest public browsing. Account actions require OAuth. A2A guest clients must retain and resend the server's guest cookie; a conversation ID alone is insufficient. Clients without cookie support should use OAuth. Guest A2A usage is limited to four messages per minute, and guest conversations expire three days after creation. After sign-in, start a new authenticated context or explicitly resume an authorized existing one; guest history does not transfer. If the guest cookie expires, discard it and send a new message without task/context IDs.

The production MCP server identifies itself as **ShopMine Production**. For testing, select **ShopMine Test** and use credentials, listing IDs and order IDs from that environment.

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
ShopMine. Saving a card is optional. Each MPP purchase requires a payment
credential for that purchase. Never collect card, bank or identity-verification
details in chat.

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
