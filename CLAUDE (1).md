# PCA Social Media Report Dashboard

Monthly social media report for agency clients, deployed to Netlify from this repo.
Every push to `main` redeploys the site automatically.

- Dashboard file: `social-report-dashboard/index.html` (one self-contained page, Chart.js from cdnjs). It always uses a white, light theme so screenshots paste cleanly into slides; don't add a dark mode.
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
   - `daily`: two series per platform, one value per day of the month. Instagram: reach and profile views. Facebook: impressions (`page_media_view`) and engagement.
     TikTok: `["Video views (by posting day)", [...], "bar"]` then `["Engagement", [...]]`. Video views come from a query with dimension `video_create_date` and metric `video_views`, filtered to videos created in the month (days with no post = 0). TikTok can't combine `date` with video metrics. Keep the TikTok `dailyNote` text.
   - `audience` (TikTok): also add `cities`: the top 5 named cities from `audience_city` / `audience_city_distribution`, skipping "Other" buckets, with the "ID " country prefix removed. The page adds "Other" itself. Leave `activeHours` as `null` unless hourly values are supplied.
   - `audience` (TikTok only): `audience_age_group` / `audience_age_distribution` and `audience_gender` / `audience_gender_distribution`, plus `cities` as described above.
   - `content`: top posts of the month sorted by views, max 15. Same row shape as the existing entries. Clean captions: drop broken emoji characters and hashtags, and keep the first sentence or two.
   - Instagram `trends` (31 values each): `reach` and `visits` (`account_reach`, `account_profile_views` by `date`); `follows` from `account_follower_count` by `date`, which only works for the last 30 days, so earlier days are `null` and `followsFrom` holds the first available day; `interactions` = `media_engagement` by `media_timestamp`, added up per posting day (Instagram can't split likes or comments by calendar day).
   - Instagram `followers`: `ageGender` = rows `[age, female, male]` for 18-24 to 55-64 from `account_followers_count` by `audience_age` + `audience_gender` (counts, not %); `cities` = top 10 `[city, count]` from `account_followers_count` by `audience_city`. Use `null` when Instagram returns nothing (small accounts).
   - Instagram `engagement`: map of post shortcode to `media_engagement`, from a query with dimension `media_permalink`. The post cards are sorted by this, so include every post in `content`.
   - Instagram thumbnails for the peak cards and post cards: download each post's image to `social-report-dashboard/thumbs/ig_<shortcode>.jpg` (shortcode from the permalink).
   - TikTok cities: TikTok only reports the top 5 cities per country, so keep 5.
   - TikTok `audience` is a snapshot of current followers: the distributions are the same whatever date range you query, so the page labels it "Followers". Call them followers, not viewers, in insights.
   - Facebook `followers`: `{ "cities": [[city, count], ...] }`, top 10 from `page_follows` by `city` (sorted desc; `city` only works with `page_follows`). Shorten names to the city ("Kota Palembang, Sumatera Selatan" → "Palembang") and add the country in brackets for non-Indonesian cities. Use `null` when Facebook returns no rows (small pages). Facebook doesn't offer follower age or gender.
   - TikTok `content` shows as cards. For each video, download its thumbnail (`video_thumbnail_url`, dimension with `video_id`) to `social-report-dashboard/thumbs/<video_id>.webp`. The URLs expire after about a day, so download them during the update, never link to them. Commit the images with the update.
   - TikTok `manual`: hand-filled per-video fields keyed by video ID: `{"cart": "Yes"/"No", "orders": number, "pillar": "..."}`. Never overwrite or delete existing entries. For new videos, leave them out (they show as "–") unless the user gives the values.
   - `insights` (inside each platform's detail): `{ "daily": [...], "audience": [...], "followers": [...], "content": [...] }`, 1–3 sentences each, shown in an "Insight" box under that section's heading. TikTok uses daily/audience/content, Instagram daily/followers/content, Facebook daily/content. Point out what the charts don't say on their own: share of the month from a few days or posts, first vs second half, reach spikes that posts don't explain, audience vs content mismatches (e.g. TikTok viewers vs Instagram followers), unusual cities, and which formats or hooks earn more likes per 1,000 views or longer watch time. Same evidence rules as takeaways.
   - `takeaways`: per platform, `{ "compare": [...], "why": [...] }` with 1–3 bullets each. The page shows them under "vs <previous month>" and "Why". A plain array still renders as one list.
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

- `compare`: what changed vs the previous month, with % change and both months' values for the key metrics.
- `why`: explain the result with evidence from the data: which posts drove it (share of views, comments/shares on giveaways), content types that over- or under-performed (watch time, likes), posting volume, organic vs total impressions, and likely paid promotion when account reach or impressions far exceed what posts reached. Mark inferences with "likely" or "points to"; don't invent causes the data can't support.
- Plain English, one sentence each, specific numbers. Say "engagement rate" with the % and the previous month's %.

## Known data context (as of October 2026)

- Leukoplast Instagram reach and Facebook impressions are driven by paid campaigns from September 2026; organic posts reach a few hundred people each.
- Fation's accounts became active in September–October 2026; months before that are `null`.
- Leukoplast's TikTok is the Essity account (`@essityindonesia`).
