# Troubleshooting

## Transport & API errors

| Error | Meaning | Fix |
|---|---|---|
| `The socket connection was closed unexpectedly` | MCP client transport blip — the request may not have reached Meta | Retry once (very common on the first call after connect/idle). For a **create**, list entities first to confirm it didn't land |
| `INTERNAL` error, `is_retryable: false`, on a create | Server-side hiccup; object state unknown | **Never blind-retry a create.** List the parent's children (e.g. `ads_get_ad_entities` level `ad`) to check for the object; if absent, retry once; if it fails twice, stop and surface |
| `VALIDATION: Unsupported field(s) at level '<x>'` | You used a Graph-API field name this MCP doesn't accept | The error's "Supported fields are: …" list is the authoritative contract — pick from it and retry |
| `This tool is new and is being gradually rolled out across ad accounts` | Per-account feature gate (seen on `ads_get_ig_accounts`) | Not fixable from code. Work without the capability and tell the user (e.g. "Facebook placements only for now; add Instagram in Ads Manager") |
| `Performance goal isn't available with this objective` | optimization_goal outside the objective's allowed list | Use the objective→goal table in tool-contracts.md |
| "Ad not found" on `ads_get_ad_preview` for a fresh ad | Draft not materialized yet | Call again with `creative_id` instead of `ad_id` |

## Images: "the files are on my computer"

`ads_creative_upload_image` fetches from a **public URL only**. For local
files, in order of preference:

1. Already uploaded before? `ads_get_ad_images` — hashes persist per account, and duplicate uploads dedupe to the same hash.
2. The user's own website/CDN — publish the image at a direct URL, upload, then it can be unpublished (Meta stores a copy).
3. Ads Manager manual upload (drag & drop), then read the hash back via `ads_get_ad_images`.
4. The Ads CLI (below) uploads local files: `meta ads creative create --image ./file.jpg …` uploads even when the creative step itself fails.

## Auth paths (decide once)

**Official MCP — `https://mcp.facebook.com/ads` (default choice).** OAuth with
the user's Facebook login; no developer app, no token management, no dev-mode
walls; creative creation just works. Interactive setup:
`claude mcp add --transport http meta-ads https://mcp.facebook.com/ads`, then
`/mcp` → authenticate (must be done by the user in an interactive session; the
login must admin the target business).

**System-user token + Meta developer app (headless/cron fallback).** Token
never expires, works in scripts. Costs: app creation ceremony and two walls
below. The Ads CLI (`pip install meta-ads` — Linux/macOS wheels only; on
Windows install inside WSL) reads `ACCESS_TOKEN` / `AD_ACCOUNT_ID` /
`BUSINESS_ID` from `.env` in the working directory.

### Wall 1: "No permissions available" when generating the system-user token

The app has no use cases, so there are no permissions to grant. App dashboard →
Use cases → add **"Create & manage ads with Marketing API"** + **"Measure ad
performance data"** (+ a Pages use case, e.g. "Manage everything on your Page").
Re-generate the token; the permission list will be populated. The system user
must also have the app assigned (Business Settings → Apps → Assign people) and
the ad account + Page as assets.

### Wall 2: "Ads creative post was created by an app that is in development mode"

Hard platform gate on creative creation for dev-mode apps — **app roles do NOT
bypass it** (verified: a system-user token with full app access still hits it).
Two real fixes:

1. Switch the app to **Live mode** — requires a **Privacy Policy URL** in App Settings → Basic first (Meta blocks the Live toggle without one). No App Review needed for managing your own business's assets.
2. Skip apps entirely: do creative work through the official MCP (the wall doesn't exist there — Meta's own first-party app is behind it).

## Business / account walls

| Wall | Fix |
|---|---|
| "Your business is prohibited from advertising, including claiming apps" | Business-level restriction (common for new/unverified portfolios). business.facebook.com/accountquality → select the Business → **Request review** (usually ID verification; minutes–48h). **Never create a parallel portfolio to route around it** — Meta links them and escalates to permanent bans. If the business shows clean, check the personal ad-access restriction on the same dashboard |
| `has_payment_method: false` | UI-only: Ads Manager → Billing → add payment method. Everything can be built before this, but nothing delivers |
| Ads stuck "In Review" | Normal for minutes–hours (sometimes 24h). Rejections come with a reason; edit and resubmit |
| `is_queryable: false` on the account | Insights tools won't work for it; surface `not_queryable_reason` |

## Georgia-specific notes

- New Georgian business portfolios frequently trip the preemptive advertising restriction — budget a day for the accountquality review before the first launch.
- Account currency is fixed at creation (USD and GEL both common). All budget integers are cents of THAT currency.
- Georgian script needs no special handling in MCP params (UTF-8 end-to-end). If scripting via CLI on Windows instead, pass Georgian text via UTF-8 script files, not shell arguments — the cmd→WSL boundary can mangle it.
- Geo radius targeting reaches people physically in the area, not "Georgians" — for Georgian speakers abroad or nationwide, discuss `countries:["GE"]` and language expectations with the user explicitly.
