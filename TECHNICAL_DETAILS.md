# Technical Details

## Why Two Tables?

- **campaign_clicks**: Raw data from CSV import
- **distributed_stats**: Your calculated results after distribution

This separation makes it easy to:
1. Re-run distribution logic without re-importing
2. Compare raw vs distributed data
3. Track what's been processed

## Integer Distribution (Important!)

When distributing integer metrics, the sum MUST equal the original.

**Example Problem:**
```
Feed: 1000 searches
Campaign A: 60% = 600 searches
Campaign B: 40% = 400 searches
Total: 1000 ✓

But what if 1001 searches?
Campaign A: 60% = 600.6 → 600
Campaign B: 40% = 400.4 → 400
Total: 1000 (missing 1!)
```

**Solution:** Largest Remainder Method
1. Calculate fractional parts for each campaign
2. Round everyone down initially
3. Distribute remaining units to campaigns with largest fractions

```python
# Example implementation
def distribute_integer(total, weights):
    distributed = [int(total * w) for w in weights]
    remainder = total - sum(distributed)

    if remainder > 0:
        fractions = [(total * w) % 1 for w in weights]
        indices = sorted(range(len(weights)),
                        key=lambda i: fractions[i],
                        reverse=True)
        for i in indices[:remainder]:
            distributed[i] += 1

    return distributed
```

## Data Relationships

```
feed_provider_data.csv          binom_clicks.csv
┌──────────────────┐            ┌──────────────────┐
│ fp_feed_id: SB100│◄──────────►│ fp_feed_id: SB100│
│ revenue: $100    │            │ campaign_id: 101 │
│ searches: 10000  │            │ clicks: 600      │
└──────────────────┘            └──────────────────┘
                                        │
                                ┌──────────────────┐
                                │ fp_feed_id: SB100│
                                │ campaign_id: 102 │
                                │ clicks: 400      │
                                └──────────────────┘
```

## FAQ / Common Questions

**Q: Why is `ts` parameter needed if API key already identifies traffic source?**
A: It's redundant but kept for backwards compatibility. Validate that `ts` matches the key's traffic_source_id.

**Q: What if multiple campaigns have zero clicks on same feed?**
A: All get zero metrics. Don't divide by zero.

**Q: Should I implement idempotency for file processing?**
A: Optional but recommended. Prevents duplicate data if script runs twice.

**Q: Which framework for API?**
A: Your choice. Flask, FastAPI, or even raw HTTP server are all fine.

## API Implementation Notes

### Authentication
```javascript
// Query parameter
const apiKey = url.searchParams.get('key');
const trafficSourceId = API_KEYS[apiKey];
```

### Data Query
```sql
SELECT date, campaign_id, campaign_name,
       total_searches, monetized_searches, paid_clicks,
       ROUND(pub_revenue, 2) as revenue,
       fp_feed_id as feed_id
FROM binom_stats
WHERE traffic_source_id = $1
  AND date >= $2 AND date <= $3
  AND is_feed_data = true
  AND pub_revenue > 0
ORDER BY date DESC, campaign_id ASC
```

### Error Handling
- 400: Invalid parameters
- 401: Invalid API key
- 403: Access denied (wrong traffic source)


## Testing Your Solution

1. Process test data
2. Check totals match
3. Verify edge cases handled
4. Test API with curl:

```bash
# Valid request
curl "http://localhost:8000/pubstats?ts=66&from=2025-01-15&to=2025-01-20&key=test_key_66"

# Invalid auth
curl "http://localhost:8000/pubstats?ts=66&from=2025-01-15&to=2025-01-20&key=wrong"

# Date range
curl "http://localhost:8000/pubstats?ts=66&from=2025-01-01&to=2025-01-31&key=test_key_66"
```