# Revenue Distribution Algorithm

## Core Concept

Distribute feed metrics (`total_searches`, `monetized_searches`, `paid_clicks`, `revenue`) from `test_searchbarista_feed_data.csv` to campaigns in `test_binom_stats_clicks.csv` based on their click contributions. SearchBarista retains a 30% commission on revenue; publishers receive 70% net revenue.

## Formula

For each `(date, feed_id)` in `feed_data`:
1. Find all campaigns in `campaign_clicks` where `feed_id` matches `feed_id` in `feed_data` (case-sensitive).
2. Calculate total clicks: `total_feed_clicks = sum(clicks)` for all matching campaigns, using `clicks` from `test_binom_stats_clicks.csv`.
3. For each campaign:
   - Proportion: `proportion = clicks / total_feed_clicks`.
   - Distributed metrics:
     - `total_searches = floor(feed_total_searches * proportion)`
     - `monetized_searches = floor(feed_monetized_searches * proportion)`
     - `paid_clicks = floor(feed_paid_clicks * proportion)`
     - `gross_revenue = feed_revenue * proportion`
     - `net_revenue = round(gross_revenue * 0.7, 4)` (store in DB)
   - Apply largest remainder method to reconcile integer metrics (`total_searches`, `monetized_searches`, `paid_clicks`).
4. API: Return `net_revenue` rounded to 2 decimals using banker’s rounding.

**Largest Remainder Method**:
- After `floor`, calculate remainder: `feed_metric - sum(floor_values)`.
- Allocate remaining units to campaigns with largest decimal remainders.
- Tiebreaker: Prioritize higher `clicks`, then lower `campaign_id`.

**Example 1: Two Campaigns**:
- Feed SB100, 2025-01-15 (`test_searchbarista_feed_data.csv`):
  - `total_searches=15420`, `monetized_searches=12336`, `paid_clicks=342`, `revenue=$125.45`
  - `total_feed_clicks=8651` from `test_binom_stats_clicks.csv` (Campaign 101: 5420 clicks, Campaign 102: 3231 clicks).
- Proportions:
  - Campaign 101: `5420 / 8651 ≈ 0.6265`
  - Campaign 102: `3231 / 8651 ≈ 0.3735`
- `total_searches`:
  - 101: `floor(15420 * 0.6265) = 9657`, remainder `0.6141`.
  - 102: `floor(15420 * 0.3735) = 5762`, remainder `0.1839`.
  - Sum floors: `9657 + 5762 = 15419`. Remaining: `15420 - 15419 = 1`.
  - 101 gets +1 (higher remainder). Final: 101=9658, 102=5762.
- `monetized_searches`:
  - 101: `floor(12336 * 0.6265) = 7726`, remainder `0.4914`.
  - 102: `floor(12336 * 0.3735) = 4610`, remainder `0.1097`.
  - Sum floors: `7726 + 4610 = 12336`. No remainder.
- `paid_clicks`:
  - 101: `floor(342 * 0.6265) = 214`, remainder `0.2653`.
  - 102: `floor(342 * 0.3735) = 127`, remainder `0.7570`.
  - Sum floors: `214 + 127 = 341`. Remaining: `342 - 341 = 1`.
  - 102 gets +1 (higher remainder). Final: 101=214, 102=128.
- Revenue:
  - 101: `$125.45 * 0.6265 = $78.60`, `net_revenue = round($78.60 * 0.7, 4) = 55.0200`
  - 102: `$125.45 * 0.3735 = $46.85`, `net_revenue = round($46.85 * 0.7, 4) = 32.7950`

**Example 2: Three Campaigns**:
- Feed SB200, 2025-01-16 (`test_searchbarista_feed_data.csv`):
  - `total_searches=10000`, `monetized_searches=8000`, `paid_clicks=200`, `revenue=$80.00`
  - `total_feed_clicks=1600` from `test_binom_stats_clicks.csv` (Campaign 22: 300 clicks, Campaign 33: 500 clicks, Campaign 44: 800 clicks).
- Proportions:
  - Campaign 22: `300 / 1600 = 0.1875`
  - Campaign 33: `500 / 1600 = 0.3125`
  - Campaign 44: `800 / 1600 = 0.5000`
- `total_searches`:
  - 22: `floor(10000 * 0.1875) = 1875`, remainder `0.0000`.
  - 33: `floor(10000 * 0.3125) = 3125`, remainder `0.0000`.
  - 44: `floor(10000 * 0.5000) = 5000`, remainder `0.0000`.
  - Sum floors: `1875 + 3125 + 5000 = 10000`. No remainder.
- `monetized_searches`:
  - 22: `floor(8000 * 0.1875) = 1500`, remainder `0.0000`.
  - 33: `floor(8000 * 0.3125) = 2500`, remainder `0.0000`.
  - 44: `floor(8000 * 0.5000) = 4000`, remainder `0.0000`.
  - Sum floors: `1500 + 2500 + 4000 = 8000`. No remainder.
- `paid_clicks`:
  - 22: `floor(200 * 0.1875) = 37`, remainder `0.5000`.
  - 33: `floor(200 * 0.3125) = 62`, remainder `0.5000`.
  - 44: `floor(200 * 0.5000) = 100`, remainder `0.0000`.
  - Sum floors: `37 + 62 + 100 = 199`. Remaining: `200 - 199 = 1`.
  - 33 gets +1 (tiebreaker: higher clicks than 22). Final: 22=37, 33=63, 44=100.
- Revenue:
  - 22: `$80.00 * 0.1875 = $15.00`, `net_revenue = round($15.00 * 0.7, 4) = 10.5000`
  - 33: `$80.00 * 0.3125 = $25.00`, `net_revenue = round($25.00 * 0.7, 4) = 17.5000`
  - 44: `$80.00 * 0.5000 = $40.00`, `net_revenue = round($40.00 * 0.7, 4) = 28.0000`

## Key Requirements
- Proportional distribution based on `clicks` from `campaign_clicks`.
- Commission (30%) applied to revenue after distribution.
- Store `net_revenue` at 4 decimals, return 2 decimals in API.
- `clicks` are internal, never exposed in API.
- Idempotent: Check `file_sha256` in `file_ingestions`.

## Edge Cases
| Scenario | Action |
|----------|--------|
| No clicks for (date, feed_id) | Skip, no allocation. |
| Clicks exist but no feed revenue | Ignore clicks, no revenue to distribute. |
| Unknown traffic_source_id | Include in calculation, store with actual ID. |
| Zero revenue in feed | Distribute integer metrics with largest remainder; revenue = 0. |
| Duplicate file upload | Check file_sha256, reject if exists. |
| Negative clicks | Skip record, log in `edge_cases_handled.md`. |
| Zero clicks | Include, allocate 0 for all metrics. |
| Duplicate feed rows | Keep one if identical, else error and skip date/feed_id. |
| Case mismatch (e.g., sb100 vs SB100) | No match, skip clicks. |

## Notes
- Dates: YYYY-MM-DD, UTC, inclusive in queries.
- Currency: USD.
- Precision: Revenue uses banker’s rounding; integers use largest remainder.