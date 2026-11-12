# PCA Social Media Report Dashboard

Monthly social media report for agency clients, deployed to Netlify from this repo.
Every push to `main` redeploys the site automatically.

- Dashboard file: `social-report-dashboard/index.html` (one self-contained page, Chart.js from cdnjs)
- All data lives in the `DATA` object inside the `<script>` block. Platform setup lives in `PLATFORMS`.
- Data source: Two Minutes Report MCP. Team ID `019d0ca3-d371-7673-a6dd-436f0ff736e2`.

## Clients and accounts

| Client (DATA key) | TikTok (`ttins`) | Instagram (`ins`) | Facebook (`fins`) |
|---|---|---|---|
| Perspirex (`perspirex`) | `-000UDNQ5c0UJ0jWiXcTID6mxItet4dmiSZS` | `17841476563577420` | `764938403367796` |
| Jordan (`jordan`) | `-00018pnFn6k4OxL-klRWPtXWxwEP8RXO9A6` | `17841426443785457` | `110753963758835` |
| Leukoplast (`leukoplast`) | `-000Su1tYjq53lZi3eo2zs0s7Uh7h1CSK_Vb` (Essity account) | `17841451812900283` | `108231665188316` |
| Fation (`fation`) | `-000-72738LE9IuxJdRdjyzQSP1EUQwLIyYZ` | `17841439368873923` | `1368652219660616` |

Only these four clients appear in the dashboard. Don't add others unless asked.

## Data model

Each client: `{ name, handles, pulled, notes, months: { "YYYY-MM": { tiktok, instagram, facebook, detail? } } }`.

Each platform value is an array in the exact metric order below, or `null` when the account had no data that month. Use `null` for missing values inside an array too. Engagement rate is calculated by the page (engagement ÷ reach); never store it.

**TikTok** (`ttins`):
1. Video views: `video_views`, videos created in the month
2. Avg. viewing duration (s, 2 decimals): `video_average_time_watched`, same filter
3. Reach: `profile_reached_audience`
4. Engagement: `profile_total_engagement`
5. New followers: `profile_new_followers`
6. Profile views: `profile_views`
7. Likes: `profile_likes`
8. Comments: `profile_comments`
9. Shares: `profile_shares`
10. Followers (end of month): `profile_current_followers`

**Instagram** (`ins`):
1. Impressions: `media_views` (posts published in the month)
2. Reach: `account_reach`
3. Engagement: `media_engagement` (posts published in the month)
4. New followers: `account_new_followers`
5. Unfollows: `account_unfollowers`
6. Profile views: `account_profile_views`
7. Engaged accounts: `profile_engaged_users`
8. Likes: `account_post_likes`
9. Comments: `account_post_comments`
10. Saves: `account_post_saves`
11. Shares: `account_post_shares`
12. Followers: `account_followers_count`, latest month only, `null` for older months

**Facebook** (`fins`):
1. Impressions: `page_media_view`
2. Organic post impressions: `page_posts_impressions_organic`
3. Reach: `page_total_media_view_unique`
4. Engagement: `page_post_engagements`
5. New followers: `page_daily_follows_unique`
6. Unfollows: `page_daily_unfollows_unique`
7. Page views: `page_views_total`
8. Video views: `page_video_views`
9. Followers: `page_followers_count`, latest month only, `null` for older months

## Monthly update ("Add <Month Year> to the dashboard")

1. For every client, run the monthly queries for the new **calendar month** (1st to last day) with `totals: true`. Never use `compareTo: "previous_period"` for stored values: it picks a rolling 30/31-day window, not the previous calendar month.
2. Add the new month block to each client, values in the metric order above.
3. Move the `detail` block to the new month (remove it from the previous month) and rebuild it:
   - `daily`: two series per platform, one value per day of the month. TikTok: profile views and engagement. Instagram: reach and profile views. Facebook: impressions (`page_media_view`) and engagement.
   - `audience` (TikTok only): `audience_age_group` / `audience_age_distribution` and `audience_gender` / `audience_gender_distribution`.
   - `content`: top posts of the month sorted by views, max 15. Same row shape as the existing entries. Clean captions: drop broken emoji characters and hashtags, and keep the first sentence or two.
   - `takeaways`: three bullets per platform.
4. Update each client's `pulled` date to today and review `notes`. Add a note when numbers need context, such as paid campaigns inflating reach or an account that is new.
5. Keep the last 6 months by default. Drop the oldest month only if asked.
6. Check the page: `node --check` on the script, then open it and click through every client, tab and the "Last 6 months" preset with no console errors.
7. Commit as `Add <Month Year> data` and push to `main`.

## Query gotchas

- **TikTok video queries** with a `video_create_date` filter drop the MCP connection when batched. Run them **one query per call**. Filter: `video_create_date greaterThan <last day of previous month>` and `lesserThan <1st of next month>`.
- **Instagram `account_follower_count`** only works with a `date` dimension and only for the last 30 days. Use `account_new_followers` / `account_unfollowers` instead.
- Batches of about 3 monthly KPI queries per call are reliable. If the connection drops, retry with smaller batches.
- Run `validate_query` before new query shapes. Never add up rows yourself; use `totals`.
- A month with all zeros before an account was active is `null`, not zeros.

## Takeaways style

- State only what the numbers show: changes vs the previous month, the top post, clear spikes in the daily data.
- No speculation about causes, except flagging likely paid promotion when account reach or impressions far exceed what posts reached.
- Plain English, one sentence each, specific numbers. Say "engagement rate" with the % and the previous month's %.

## Known data context (as of October 2026)

- Leukoplast Instagram reach and Facebook impressions are driven by paid campaigns from September 2026; organic posts reach a few hundred people each.
- Fation's accounts became active in September–October 2026; months before that are `null`.
- Leukoplast's TikTok is the Essity account (`@essityindonesia`).
