# meta-ads — a Claude skill for running Meta (Facebook/Instagram) ads

Launch and manage real Meta ad campaigns by talking to Claude — through Meta's
**official Ads MCP server** (`mcp.facebook.com/ads`). No developer app, no
token wrangling, no Ads Manager spelunking. Born from a real production launch
in Tbilisi 🇬🇪 and battle-tested against every wall Meta put up along the way.

[ქართული ვერსია ქვემოთ ⬇](#ქართულად)

## What it gives Claude

- The **verified tool contracts** of the Ads MCP (they differ from the Graph API in call-breaking ways — parameter names, JSON-string params, field lists)
- An end-to-end **launch playbook**: account checks → campaign → ad set → creatives → ads → previews → activation, with the two mandatory user-confirmation pauses
- **Georgia-ready recipes**: Tbilisi radius targeting without city-key lookups, country targeting, Georgian-language ad copy, currency/cents traps
- A **troubleshooting playbook** for the real errors: socket drops, INTERNAL create failures, dev-mode walls, "prohibited from advertising" appeals, token-permission dead ends

## Install

1. **The skill** — copy this folder into your Claude Code skills directory:

   ```bash
   git clone https://github.com/sepivip/meta-ads-skill ~/.claude/skills/meta-ads
   ```

   (Windows: `git clone https://github.com/sepivip/meta-ads-skill "$env:USERPROFILE\.claude\skills\meta-ads"`)

   Downloaded the ZIP instead (e.g. from Agensi)? Unzip it into your skills
   directory so the folder lands at `~/.claude/skills/meta-ads/` — the folder
   name must stay `meta-ads`.

2. **The MCP server** — register and authenticate (one time, interactive):

   ```bash
   claude mcp add --transport http meta-ads https://mcp.facebook.com/ads
   ```

   Then in Claude Code type `/mcp`, pick `meta-ads`, and log in with the
   Facebook account that administers your business.

3. Ask Claude to launch a campaign. It will build everything **paused**, show
   you previews, and activate only when you say go.

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

## ქართულად

ეს არის Claude-ის სკილი Meta-ს (Facebook/Instagram) რეკლამების სამართავად —
Meta-ს **ოფიციალური Ads MCP სერვერით** (`mcp.facebook.com/ads`). არ გჭირდება
დეველოპერული აპლიკაცია, ტოკენების გენერაცია ან Ads Manager-ში ხეტიალი:
კამპანიას Claude-თან საუბრით უშვებ.

შექმნილია თბილისში რეალური კამპანიის გაშვების გამოცდილებაზე — ყველა კედელი,
რომელსაც Meta ახალ რეკლამდამკვეთს უღობავს (ბიზნესის შეზღუდვის მოხსნა,
ნებართვები, dev-mode-ის შეცდომები), აქ უკვე გავლილი და დოკუმენტირებულია.

**რას შეძლებ:**

- კამპანიის სრული გაშვება: ბიუჯეტი, თბილისის (ან მთელი საქართველოს) თარგეთინგი, ქართულენოვანი კრეატივები, გააქტიურება — ყველაფერი Claude-ის საშუალებით
- შედეგების მონიტორინგი: რომელი კრეატივი მუშაობს, CTR, კლიკის ფასი, დახარჯული თანხა
- პრობლემების მოგვარება: „prohibited from advertising", „development mode", გადახდის მეთოდი და სხვა ტიპური კედლები

**უსაფრთხოება:** ყველაფერი იქმნება გაჩერებულ (PAUSED) მდგომარეობაში — თანხა
არ იხარჯება, სანამ თავად არ დაადასტურებ გაშვებას. სატესტო კამპანიებზე
გამოიყენე lifetime ბიუჯეტი — ის ხარჯვის სტრუქტურული ჭერია.

**ინსტალაცია:** იხილე ინგლისური ინსტრუქცია ზემოთ (Install) — ორი ნაბიჯია:
სკილის დაკოპირება და MCP სერვერის დამატება `/mcp` ავტორიზაციით.

---

MIT License. Built with Claude Code. PRs welcome — especially more Georgian
targeting recipes and error signatures.
