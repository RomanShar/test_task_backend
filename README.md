# Backend Developer Test Assignment

**Time:** 6-8 hours | **Deadline:** 48 hours

## Task Overview

Build a data pipeline that distributes advertising revenue from feed providers to publisher campaigns based on their traffic contribution.

## Business Context

**Feed providers** pay for search traffic. This revenue needs to be distributed to **publisher campaigns** that generated the traffic.

You have two data sources:
1. **test_clicks.csv** - Campaign traffic data (campaign_id, fp_feed_id, clicks)
2. **test_feeds.csv** - Feed provider payments (fp_feed_id, revenue, searches)

Your job: Match feeds to campaigns via `fp_feed_id`, distribute metrics proportionally by clicks, serve via API.

## Quick Example

```
Feed SB100 earned $100 on 2025-01-15
├── Campaign 101: 600 clicks (60%) → Gets $45 (60% × $100 × 75%)
└── Campaign 102: 400 clicks (40%) → Gets $30 (40% × $100 × 75%)

Note: Publishers get 75%, platform keeps 25% commission
```

## Database Schema

```sql
-- Raw campaign data from test_clicks.csv
CREATE TABLE campaign_clicks (
    date DATE NOT NULL,
    campaign_id INTEGER NOT NULL,
    campaign_name VARCHAR(200),
    fp_feed_id VARCHAR(20),
    traffic_source_id INTEGER,
    clicks INTEGER,
    PRIMARY KEY (date, campaign_id)
);

-- Distributed metrics (your calculated results)
CREATE TABLE distributed_stats (
    date DATE NOT NULL,
    campaign_id INTEGER NOT NULL,
    campaign_name VARCHAR(200),
    fp_feed_id VARCHAR(20),
    traffic_source_id INTEGER,
    total_searches INTEGER,
    monetized_searches INTEGER,
    paid_clicks INTEGER,
    pub_revenue DECIMAL(10,2),
    PRIMARY KEY (date, campaign_id)
);

-- API authentication
CREATE TABLE publisher_keys (
    api_key VARCHAR(50) PRIMARY KEY,
    traffic_source_id INTEGER NOT NULL
);

INSERT INTO publisher_keys VALUES
    ('test_key_66', 66),
    ('test_key_67', 67),
    ('test_key_68', 68);
```

## Processing Algorithm

For each feed on each date:
1. Find all campaigns with matching `fp_feed_id`
2. Calculate each campaign's share: `weight = campaign_clicks / total_clicks`
3. Distribute metrics: `campaign_metric = feed_metric × weight`
4. Calculate revenue: `pub_revenue = feed_revenue × weight × 0.75`
5. Store in `distributed_stats` table

## API Specification

### Endpoint
```
GET /pubstats?ts={traffic_source_id}&from={date}&to={date}&key={api_key}
```

### Response (Example)
```json
[
  {
    "date": "2025-01-15",
    "campaign_id": 101,
    "campaign_name": "US_Search_Mobile_1",
    "total_searches": <calculated>,    // Proportionally distributed
    "monetized_searches": <calculated>,
    "paid_clicks": <calculated>,
    "revenue": <calculated>,            // pub_revenue, 2 decimals
    "feed_id": "SB100"
  }
]
```

### Requirements
- Authenticate via API key
- Return only records where `pub_revenue > 0`
- Sort by date DESC, campaign_id ASC
- Max date range: 90 days
- Return `[]` for no data

## Test Data Edge Cases

Your solution must handle these scenarios found in the test data:

1. **Duplicate feed entries**: Same date + fp_feed_id appears twice
   - Solution: Use the last occurrence

2. **No matching campaigns**: Feed exists but no campaigns have that fp_feed_id
   - Solution: Skip this feed, log warning

3. **Negative clicks**: Campaign has clicks = -100
   - Solution: Exclude from calculation

4. **Case sensitivity**: 'SB100' vs 'sb100'
   - Solution: Treat as different IDs

5. **Zero clicks**: Campaign exists with 0 clicks
   - Solution: Include with all metrics = 0

## Deliverables

```
your-solution/
├── README.md           # How to run
├── schema.sql          # Database setup
├── process.py          # Data processing
├── api.py              # API server
└── tests.py            # Basic tests
```

## Evaluation

We'll check:
1. Correct proportional distribution
2. 75% revenue share applied
3. API authentication works
4. Edge cases handled gracefully
5. Clean, maintainable code

Good luck! 🚀