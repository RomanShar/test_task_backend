# Revenue Distribution Algorithm

## Core Concept

You need to implement a proportional distribution system for ALL metrics:

1. **Match feed data to campaigns** - Use the relationship between Feed_id and tracking_tag
2. **Distribute ALL metrics proportionally** - Based on click ratios:
   - Revenue (then apply 30% commission)
   - Total searches
   - Monetized searches
   - Paid clicks (ad clicks)
3. **Apply commission** - Platform keeps 30%, publishers receive 70% of revenue ONLY
4. **Handle precision**:
   - Money: Store 4 decimals in DB, display 2 decimals in API
   - Integer metrics: Use largest remainder method for reconciliation

## Key Requirements

- Distribution must be proportional to clicks within each (date, feed) group
- All feed metrics (searches, paid_clicks, revenue) distributed by same proportion
- Commission (30%) applied ONLY to revenue AFTER distribution
- Integer metrics reconciled using largest remainder method
- Integer metrics are distributed regardless of revenue; revenue may be zero
- Clicks are internal calculation only - NEVER exposed in API
- Process must be idempotent

## Implementation Hints

Think about:
- How to handle feeds with no matching clicks
- What happens when clicks exist but no revenue
- Proper decimal handling for financial calculations
- Case sensitivity in string matching

## Edge Cases (MUST HANDLE)

| Scenario | Action |
|----------|--------|
| No clicks for (date, feed_id) | Skip - no revenue allocated |
| Clicks exist but no feed revenue | Ignore clicks - no revenue to distribute |
| Unknown traffic_source_id | Include in calculation, store with actual ID (company name lookup is out of scope) |
| Zero revenue in feed | Distribute integer metrics (searches, paid_clicks) with largest remainder; revenue = 0 |
| Duplicate file upload | Check file hash, reject if exists |

## Data Handling Rules

1. **Date format**: YYYY-MM-DD (dates are inclusive in queries)
2. **String matching**: Feed_id == tracking_tag (case-sensitive, exact match)
3. **Precision**: Store at 4 decimals in DB, return 2 decimals in API by default
   - Use "largest remainder" method for reconciliation when rounding to 2 decimals
   - Sum of rounded amounts must equal round(total * 0.70, 2)
   - Tiebreaker for equal fractional parts: prioritize campaigns with more clicks, then by ascending campaign_id
4. **Currency**: All amounts in USD
5. **Timezone**: All dates in UTC (no time component)
6. **Negative clicks**: Skip records with negative clicks and log in edge_cases_handled.md
7. **Duplicate rows**: Exact duplicates - keep one; conflicting duplicates - reject with error

## Idempotency Key

Natural key for deduplication:
```sql
UNIQUE(date, campaign_id, feed_id)
```

## Example Scenario

**Given:**
- Feed SB100 has revenue on 2025-01-15
- Multiple campaigns used tracking_tag "SB100" that day
- Each campaign has different click counts

**Your task:**
Calculate how much each campaign should receive based on their click share, after applying the platform commission.

**Think about:**
- What's the mathematical formula for proportional distribution?
- At what point do you apply the 30% commission?
- How do you handle rounding for currency amounts?