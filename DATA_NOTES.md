# Data Notes for Testing

## ⚠️ Important

The test data contains several intentional anomalies and edge cases to test your implementation. Not all data is "clean" - you must handle these cases appropriately.

## Known Issues in Test Data & How to Handle

1. **Duplicate Records** - Feed data contains duplicate rows
   - Exact duplicates: Keep only one
   - Conflicting duplicates (same date/feed_id, different revenue): Reject with error

2. **Case Sensitivity** - Case mismatches between Feed_id and tracking_tag
   - Use strict case-sensitive matching
   - 'sb100' ≠ 'SB100' - no match, no revenue distribution

3. **Zero or Negative Clicks**
   - Zero clicks: Include in processing but no revenue allocation
   - Negative clicks: Skip the record and document in edge_cases_handled.md
   - Note: campaign_clicks = internal metric for proportion calculation (never exposed in API)
   - Note: paid_clicks = distributed feed metric (exposed to publishers in API)

4. **Missing Matches** - Not all clicks have corresponding feed revenue
   - Clicks without feed revenue: Ignore (no revenue to distribute)
   - Feed revenue without clicks: Skip (no distribution possible)

5. **Rounding & Precision**
   - feed_data.revenue: stored as in source file (typically 2 decimals)
   - revenue_allocations.net_revenue: stored at 4 decimal precision
   - API returns 2 decimals by default (optional ?precision=4)
   - Integer metrics (searches, paid_clicks): Use "largest remainder" for reconciliation
   - ALL metrics distributed by same proportion (campaign_clicks / total_clicks)
   - Clicks are internal only - NEVER expose in API

## Validation Data

VALIDATION_DATA.csv contains **sample calculations only**. It is intentionally incomplete and should be used to verify your calculation approach, not as a complete answer key.

## What We're Testing

- Your ability to identify and handle data quality issues
- Proper error handling and edge case management
- Understanding of the business logic beyond just copying formulas
- Your approach to dealing with ambiguous situations

## Important: Provider vs Publisher Perspective

Providers → input only, Publishers → API output. Never expose feed_id or raw clicks in API.

## Hint

If something seems wrong in the data, it probably is. Document how you handle it in your edge_cases_handled.md file.