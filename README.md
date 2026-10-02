# meta-ads — a Claude skill for running Meta (Facebook/Instagram) ads

Launch and manage real Meta ad campaigns by talking to Claude — through Meta's
**official Ads MCP server** (`mcp.facebook.com/ads`). No developer app, no
token wrangling, no Ads Manager spelunking. Born from a real production launch
and battle-tested against every wall Meta put up along the way.

## What it gives Claude

- The **verified tool contracts** of the Ads MCP (they differ from the Graph API in call-breaking ways — parameter names, JSON-string params, field lists)
- An end-to-end **launch playbook**: account checks → campaign → ad set → creatives → ads → previews → activation, with the two mandatory user-confirmation pauses
- **Targeting recipes**: radius targeting around any point without city-key lookups, country targeting, ad copy in any language or script, currency/cents traps
- A **troubleshooting playbook** for the real errors: socket drops, INTERNAL create failures, dev-mode walls, "prohibited from advertising" appeals, token-permission dead ends

## Install

1. **The skill** — put the `meta-ads` folder in your skills directory, so
   `SKILL.md` ends up at `~/.claude/skills/meta-ads/SKILL.md`
   (Windows: `%USERPROFILE%\.claude\skills\meta-ads\SKILL.md`). Keep the
   folder name `meta-ads`.

   - **From the ZIP** (e.g. Agensi): unzip it into `~/.claude/skills/`.
   - **From git:**

     ```bash
     git clone https://github.com/sepivip/meta-ads-skill ~/.claude/skills/meta-ads
     ```

2. **The MCP server** — register and authenticate (one time, interactive):

   ```bash
   claude mcp add --transport http meta-ads https://mcp.facebook.com/ads
   ```

   Then in Claude Code type `/mcp`, pick `meta-ads`, and log in with the
   Facebook account that administers your business.

3. Ask Claude to launch a campaign. It will build everything **paused**, show
   you previews, and activate only when you say go.

## Try it

Paste one of these into Claude after setup:

- *"Check my Meta ad accounts — which ones are ready to run ads?"*
- *"Launch a traffic campaign for https://example.com: $100 total over 2 weeks, 25 km around our shop, ages 25–55, these 3 image URLs and headlines. Show me previews before anything goes live."*
- *"How are my ads doing this week? Rank them by spend and tell me which creative is winning."*
- *"Pause the worst-performing ad in my launch campaign."*
- *"Meta says my business is prohibited from advertising — what do I do?"*

## Safety model

- Everything is created PAUSED; nothing spends until you explicitly approve activation.
- Prefer **lifetime budgets** for tests — a structural spend cap, not a promise.
- Claude checks your account has a payment method and MCP access before building anything.

## Network access

The skill is instructions only — no scripts, and it makes no network calls of
its own. It teaches Claude to use Meta's official Ads MCP server
(`https://mcp.facebook.com/ads`), which you connect and authenticate yourself
with your Facebook login. The optional headless fallback, Meta's official Ads
CLI (`pip install meta-ads`), talks to the Meta Marketing API with a token you
keep in a local `.env`. The skill sends nothing anywhere else.

## Requirements

- Claude Code (or any MCP-capable agent) · a Facebook Page · a Meta ad account with a payment method · admin access to the business portfolio

---

MIT License. Built with Claude Code. PRs welcome — especially more
targeting recipes and error signatures.
