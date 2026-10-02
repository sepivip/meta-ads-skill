# Verified MCP tool contracts

Verified against the live `mcp.facebook.com/ads` schemas (July 2026). Parameter
names below are exact. Where a param takes "JSON", it means a **JSON string**,
not a nested object.

## ads_get_ad_accounts

No required params. Each entry returns `ad_account_id` (numeric, no `act_`
prefix — all other tools want this numeric form), `ad_account_name`,
`business_id`, `business_name`, `account_status`, `currency`,
`has_payment_method`, `is_ads_mcp_enabled`, `is_queryable`,
`min_daily_budget_cents`. Gates:

- `is_ads_mcp_enabled: false` → do not use that account with these tools.
- `is_queryable: false` → do not call `ads_get_ad_entities` on it; surface `not_queryable_reason`.
- `has_payment_method: false` → structure can be created but nothing will deliver; tell the user to add one in Ads Manager (UI-only).

## ads_create_campaign

Required: `ad_account_id`, `campaign_name`, `objective`, `buying_type`.

- `objective` — ODAX only: `OUTCOME_AWARENESS | OUTCOME_TRAFFIC | OUTCOME_ENGAGEMENT | OUTCOME_LEADS | OUTCOME_SALES | OUTCOME_APP_PROMOTION`. Legacy values (LINK_CLICKS, REACH, …) are rejected with VALIDATION.
- `buying_type` — `AUCTION` for normal campaigns. Required; omitting it is a call-breaking mistake.
- Budget mode decision happens HERE:
  - **CBO (recommended default):** set `campaign_daily_budget` OR `campaign_lifetime_budget` (integer, cents of account currency). Lifetime requires `campaign_stop_time` (ISO 8601). `campaign_bid_strategy` defaults to `LOWEST_COST_WITHOUT_CAP`.
  - **ABO:** set NO budget fields here; put `daily_budget`/`lifetime_budget` on the ad set. Only when the user explicitly wants per-ad-set budgets.
- Creates PAUSED. Response includes `valid_optimization_goals` and `recommended_optimization_goal` — use them for the ad set.

## ads_create_ad_set

Required: `ad_account_id`, `campaign_id`, `ad_set_name`, `billing_event`,
`optimization_goal`, `targeting`.

- `targeting` — **JSON string.** Broad geo targeting is the recommended shape. Never invent interest IDs (real ones are 13–16 digit numbers from Meta's Targeting Search; placeholders are rejected).
- `billing_event` — usually `IMPRESSIONS`.
- Budget: ONLY in ABO mode (parent campaign has no budget). `lifetime_budget` requires `end_time`. Under a CBO parent, passing any budget/bid field here is rejected ("Must Use Campaign Bid Strategy" class of errors).
- Even under CBO with a lifetime budget, give the ad set `start_time`/`end_time` matching the flight.
- **Advantage+ Audience is ON by default**: `age_min`/`age_max` become suggestions, not caps. For hard caps include `"targeting_automation":{"advantage_audience":0}` in the targeting JSON.
- EU note: if `geo_locations.countries` includes an EU country, DSA fields apply (auto-filled from the business name; override with `dsa_beneficiary`/`dsa_payor`). Non-EU targeting (e.g. US, GB) doesn't need them.

### objective → allowed optimization_goal (default first)

| Objective | Allowed goals |
|---|---|
| OUTCOME_AWARENESS | REACH, IMPRESSIONS, AD_RECALL_LIFT, THRUPLAY, TWO_SECOND_CONTINUOUS_VIDEO_VIEWS |
| OUTCOME_TRAFFIC | LINK_CLICKS, LANDING_PAGE_VIEWS, OFFSITE_CONVERSIONS, IMPRESSIONS, POST_ENGAGEMENT, REACH, CONVERSATIONS, THRUPLAY, VISIT_INSTAGRAM_PROFILE, PROFILE_VISIT, QUALITY_CALL, REMINDERS_SET |
| OUTCOME_ENGAGEMENT | THRUPLAY, POST_ENGAGEMENT, EVENT_RESPONSES, PAGE_LIKES, IMPRESSIONS, REACH, TWO_SECOND_CONTINUOUS_VIDEO_VIEWS, VIDEO_VIEWS, LINK_CLICKS, CONVERSATIONS, OFFSITE_CONVERSIONS, LANDING_PAGE_VIEWS, QUALITY_CALL |
| OUTCOME_LEADS | OFFSITE_CONVERSIONS, LEAD_GENERATION, QUALITY_LEAD, LANDING_PAGE_VIEWS, LINK_CLICKS, IMPRESSIONS, REACH, VALUE, CONVERSATIONS, QUALITY_CALL |
| OUTCOME_SALES | OFFSITE_CONVERSIONS, VALUE, LANDING_PAGE_VIEWS, IMPRESSIONS, POST_ENGAGEMENT, REACH, LINK_CLICKS, CONVERSATIONS |
| OUTCOME_APP_PROMOTION | APP_INSTALLS, OFFSITE_CONVERSIONS, IMPRESSIONS, LINK_CLICKS, REACH, VALUE, VIDEO_VIEWS |

`LANDING_PAGE_VIEWS` and conversion goals want a `promoted_object` (pixel);
without a pixel on the site, `LINK_CLICKS` is the safe traffic goal.

## ads_creative_upload_image

Required: `ad_account_id`, `image_url`. **The server fetches the image from a
public URL — there is no local-file upload in the MCP.** JPEG/PNG/GIF, ≤30 MB.
Duplicate content returns the existing hash. Google Drive/Dropbox share links
fail (interstitial HTML); use a direct CDN/website URL. For local files see
troubleshooting §Images. Already-uploaded images: `ads_get_ad_images`
(listing returns only `hash` + `name`; re-query with `hashes` for other
fields).

## ads_create_creative

Required: `ad_account_id`, `page_id`. Image ads also need `link_url` and
exactly one of `image_hash` | `image_url`.

**Flat parameters — do NOT build `object_story_spec` yourself:**
`name`, `message` (body text), `headline`, `description`,
`call_to_action_type` (UPPER_CASE enum, defaults LEARN_MORE),
`instagram_user_id` (omit → no Instagram delivery), `video_id` (video ads),
`cards` (static carousel, 2–10), `product_set_id` (catalog carousel).
UTM tags: append them to `link_url` directly (no separate url_tags param).

## ads_create_ad

Required: `ad_account_id`, `ad_set_id`, `ad_name` (NOT `name`).
`creative` — **JSON string**, e.g. `"{\"creative_id\":\"123\"}"`. Alternatives
inside that JSON: `object_story_id` ("pageID_postID" to boost a post) or an
inline `object_story_spec` (must contain `page_id`). Creates PAUSED.

## ads_activate_entity

Required: `ad_account_id`, `entity_id`, `entity_type` ∈ `campaign | ad_set | ad`
(**`ad_set` with underscore**). No `status` param. Activating a parent does NOT
activate children — go top-down and activate every level; ads deliver only when
all three levels are ACTIVE. Only call after the user explicitly confirms going
live. Pausing back is `ads_update_entity` with a status field, not this tool.

## ads_get_ad_entities (insights + attributes)

Required: `ad_account_id`, `fields`. Key rules:

- `level`: `ad_account | campaign | adset | ad` (note: here it IS `adset`, no underscore — only `ads_activate_entity` uses `ad_set`).
- **Metrics require `date_preset` or `time_range`** (never both). Without one you get attributes only. Presets include `today, yesterday, last_3d, last_7d, last_14d, last_30d, last_90d, this_month, last_month, maximum`.
- `sort`: `<field>_ascending|_descending` (e.g. `amount_spent_descending`). Not supported at account level.
- One `breakdowns` value max per call.

### Supported ad-level fields (authoritative list, from the API's own error)

`name`, `id`, `status`, `effective_status`, `delivery`, `objective`,
`adset_id`, `campaign_id`, `creative_id`, `created_time`, `updated_time`,
`impressions`, `reach`, `frequency`, `clicks`, `amount_spent`, `ctr`, `cpc`,
`cpm`, `cpp`, `results`, `cost_per_result`, `outbound_clicks`,
`outbound_clicks_ctr`, `landing_page_view`, `post_engagement`, `lead`,
`cost_per_lead`, `unique_link_click`, `unique_link_clicks_ctr`, `website_ctr`,
`actions:link_click`, `actions:like`, `actions:comment`,
`actions:post_reaction`, `actions:post_save`, `actions:page_engagement`,
`actions:omni_purchase`, `cost_per_link_click`, `cost_per_action_type`,
`cost_per_outbound_click`, `cost_per_thruplay`, `video_play_actions`,
`video_thruplay_watched_actions`, `video_p25/50/75/95/100_watched_actions`,
purchase/ROAS fields (`purchase_roas`, `omni_*`, `offsite_conversion_*`).

**Traps:** it is `amount_spent` not `spend`*, `name` not `ad_name`, and link
clicks are the direct field `actions:link_click` (bare `actions` is invalid —
this MCP is not the raw Insights API). *`spend` in a request happens to be
normalized to `amount_spent` today, but use the canonical name — `sort`
requires it.

## ads_get_ad_preview

Provide `ad_id` OR `creative_id` (fresh draft ads sometimes 404 — fall back to
`creative_id`). `ad_format` ∈ `DESKTOP_FEED_STANDARD, MOBILE_FEED_STANDARD,
INSTAGRAM_STANDARD, INSTAGRAM_STORY, INSTAGRAM_REELS, RIGHT_COLUMN_STANDARD,
MESSENGER_MOBILE_INBOX_MEDIA, THREADS_STREAM`. Always give the user the
returned `preview_url` as a clickable link.

## Async reports

`ads_entity_schedule_report` → `report_run_id` → `ads_entity_get_report`
(long-polls ~25s internally; don't busy-retry; page via `next_cursor`). Use for
big pulls; `ads_get_ad_entities` is fine for ≤ hundreds of rows.
