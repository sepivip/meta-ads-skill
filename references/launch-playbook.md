# End-to-end launch playbook

A worked, real-shaped sequence (from a production launch: 5-creative traffic
campaign, $100 lifetime, city radius targeting). Adapt values; keep the order
and the two pauses.

## 0. Preconditions

- MCP connected (`claude mcp add --transport http meta-ads https://mcp.facebook.com/ads`, then `/mcp` → authenticate with the Facebook login that admins the business).
- `ads_get_ad_accounts` → pick the account; verify `is_ads_mcp_enabled`, `has_payment_method`, note `currency` (budgets will be in ITS cents — a "$100" ask against a EUR account is a conversion conversation with the user, not a relabel).
- `ads_get_ad_account_pages` → the Page the ads publish under.
- Images in the account library? `ads_get_ad_images` → hashes. If not: `ads_creative_upload_image` needs a **public URL** (see troubleshooting §Images for local files).

## PAUSE 1 — confirm with the user

Account + Page identity, budget + duration, targeting, landing URL, ad copy
(never invent brand voice silently), CTA choice. Cheap to ask now, expensive
after launch.

## 1. Campaign — one CBO campaign, budget lives here

```json
{
  "ad_account_id": "1234567890",
  "campaign_name": "Example Bakery - London Traffic Launch",
  "objective": "OUTCOME_TRAFFIC",
  "buying_type": "AUCTION",
  "campaign_lifetime_budget": 10000,
  "campaign_start_time": "2026-07-23T16:00:00+0100",
  "campaign_stop_time": "2026-08-12T23:59:00+0100"
}
```

`10000` = $100.00. Lifetime budget = hard structural cap AND even pacing across
the flight — the right default for a fixed test budget ("spend $X total").
Daily budget = ongoing campaigns. One campaign + one ad set + N ads (one per
creative) lets Meta's delivery shift budget to the winning creative — the
cheapest possible creative test. Only split ad sets when audiences differ.

## 2. Ad set — targeting, no budget (CBO parent)

```json
{
  "ad_account_id": "1234567890",
  "campaign_id": "<from step 1>",
  "ad_set_name": "London 18-55 - 25km",
  "optimization_goal": "LINK_CLICKS",
  "billing_event": "IMPRESSIONS",
  "start_time": "2026-07-23T16:00:00+0100",
  "end_time": "2026-08-12T23:59:00+0100",
  "targeting": "{\"geo_locations\":{\"custom_locations\":[{\"latitude\":51.5074,\"longitude\":-0.1278,\"radius\":25,\"distance_unit\":\"kilometer\"}]},\"age_min\":18,\"age_max\":55,\"targeting_automation\":{\"advantage_audience\":0}}"
}
```

- `custom_locations` lat/lng+radius avoids the city-key lookup problem entirely (there is no targeting-search tool in the MCP). Whole country: `{"countries":["GB"]}`.
- `LINK_CLICKS` unless the site has a Meta Pixel (then `LANDING_PAGE_VIEWS` + `promoted_object` pixel is better traffic quality).
- `advantage_audience: 0` makes 18–55 a hard cap; omit it and ages are suggestions.
- Geo targets people IN the area (incl. tourists/expats), not nationality or language — say so when the user asks "does this target Spanish speakers?".

## 3. Creatives — one per image, flat params

```json
{
  "ad_account_id": "1234567890",
  "page_id": "111222333444555",
  "name": "Feed 1 - Fresh bread",
  "image_hash": "<from ads_get_ad_images / upload>",
  "link_url": "https://example.com/ka?utm_source=facebook&utm_medium=paid&utm_campaign=london-launch&utm_content=feed-1",
  "message": "Fresh bread and pastries, baked every morning.",
  "headline": "Hot bread from 8am",
  "call_to_action_type": "LEARN_MORE"
}
```

- Any language or script passes through cleanly (UTF-8 end-to-end) — non-Latin copy needs no escaping.
- UTMs go inside `link_url`; vary `utm_content` per creative so external analytics can separate variants.
- Keep the CTA identical across variants when the goal is comparing creatives — vary one thing at a time.
- Omit `instagram_user_id` → Facebook-only delivery. (`ads_get_ig_accounts` is rollout-gated; if unavailable, launch FB-only and add IG later in Ads Manager.)

## 4. Ads — creative as a JSON string, into the one ad set

```json
{
  "ad_account_id": "1234567890",
  "ad_set_id": "<from step 2>",
  "ad_name": "Feed 1 - Fresh bread",
  "creative": "{\"creative_id\":\"<from step 3>\"}"
}
```

On INTERNAL error here: list ads in the ad set first (was it created anyway?),
then retry once. Verified pattern: one such error, object NOT created, retry
succeeded.

## 5. Preview + PAUSE 2 — final go/no-go

`ads_get_ad_preview` per ad (`MOBILE_FEED_STANDARD`), give the user the
`preview_url` links + a summary (budget, dates, targeting, landing URLs).
**Do not activate until the user explicitly says go.**

## 6. Activate — top-down, every level

`ads_activate_entity` × (1 campaign, then 1 ad set with `entity_type:"ad_set"`,
then EVERY ad). Then set expectations: Meta ad review (minutes–hours) before
delivery; lifetime budget paces automatically; delivery stops at end_time.

## 7. Monitor

Per-ad scoreboard (run any time):

```json
{
  "ad_account_id": "1234567890",
  "level": "ad",
  "fields": ["id","name","status","impressions","clicks","amount_spent","ctr","cpc","actions:link_click","cost_per_link_click"],
  "date_preset": "maximum",
  "sort": "amount_spent_descending"
}
```

Reading it: Meta concentrates spend on its predicted winner early — a
low-spend variant with the best CTR may be under-tested, not bad. Give the
flight time; pause only clear laggards (high spend share + worst CTR + worst
CPC) via `ads_update_entity`, and report pacing honestly (spent% vs elapsed%).
Campaign totals: same call at `level: "campaign"`.
