# Backend Developer Test Assignment

**Time:** 6-7 hours | **Deadline:** 48 hours

## Task Overview

Build a data pipeline that:
1. Processes two CSV files (feed revenue + campaign clicks)
2. Distributes revenue to campaigns proportionally by clicks
3. Serves aggregated data via simple HTTP API

## Core Requirements (4-5 hours)

### 1. Database Schema (30 min)

Create these Supabase tables:

```sql
-- Raw feed data
CREATE TABLE feed_data (
    date DATE NOT NULL,
    feed_id VARCHAR(10) NOT NULL,
    total_searches INTEGER,
    monetized_searches INTEGER,
    paid_clicks INTEGER,
    revenue DECIMAL(10,2),
    PRIMARY KEY (date, feed_id)
);

-- Campaign clicks data
CREATE TABLE campaign_clicks (
    date DATE NOT NULL,
    campaign_id INTEGER NOT NULL,
    tracking_tag VARCHAR(10) NOT NULL,
    clicks INTEGER,
    traffic_source_id INTEGER,
    campaign_name VARCHAR(100),
    PRIMARY KEY (date, campaign_id)
);

-- Calculated distributions (all metrics)
CREATE TABLE revenue_allocations (
    date DATE NOT NULL,
    campaign_id INTEGER NOT NULL,
    feed_id VARCHAR(10) NOT NULL,
    traffic_source_id INTEGER,
    campaign_name VARCHAR(100),
    -- Internal calculation only, never exposed:
    clicks INTEGER,
    -- Distributed metrics visible to publishers:
    total_searches INTEGER,
    monetized_searches INTEGER,
    paid_clicks INTEGER,                -- Ad clicks (proportionally distributed)
    net_revenue DECIMAL(10,4),         -- After 30% commission (publisher's share)
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (date, campaign_id, feed_id)
);

-- File ingestion tracking for idempotency
CREATE TABLE file_ingestions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_name TEXT NOT NULL,
    file_sha256 TEXT NOT NULL,
    rows_ingested INTEGER NOT NULL,
    ingested_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(file_sha256)
);

-- Create indexes
CREATE INDEX idx_allocations_source ON revenue_allocations(traffic_source_id, date);
CREATE INDEX idx_allocations_date ON revenue_allocations(date);
CREATE INDEX idx_feed_data_date ON feed_data(date, feed_id);
CREATE INDEX idx_clicks_tracking ON campaign_clicks(date, tracking_tag);
```

### 2. Data Processing Script (2 hours)

Create `process_data.py` that:

```python
# 1. Load both CSV files
# 2. For each (date, feed_id) from feed CSV:
#    - Find matching clicks where tracking_tag == feed_id
#    - Calculate click proportions for each campaign
#    - Distribute ALL metrics proportionally:
#      * total_searches, monetized_searches, paid_clicks
#      * revenue (then apply 30% commission for net_revenue)
#    - Use largest remainder for integer reconciliation
#    - Insert into revenue_allocations

# See DISTRIBUTION_ALGORITHM.md for concept & constraints
# You must formalize the math yourself and document it in calculation_explanation.md
```

**Key test cases to verify:**
- Feed SB100 on 2025-01-15: $125.45 revenue to distribute
- Multiple campaigns can share same feed (check tracking_tag)
- Apply proportional distribution based on clicks
- Remember 30% commission (publishers get 70%)

### 3. HTTP API Endpoint (1.5 hours)

Create an HTTP endpoint using one of these options:

#### Option A: Supabase Edge Function (Recommended)
Create Edge Function `pubstats`:

**Request:**
```
GET /functions/v1/pubstats?ts=66&from=2025-01-15&to=2025-01-20&key=test_key_66
```

#### Option B: FastAPI or Other Framework
If you prefer not to use Supabase Edge Functions, you can implement using FastAPI, Express, Flask, etc.:

**Request:**
```
GET /pubstats?ts=66&from=2025-01-15&to=2025-01-20&key=test_key_66
```

**Note:** The API contract (parameters, response format, authentication) must remain IDENTICAL regardless of implementation choice.

**Response Structure:**
```json
[
  {
    "date": "2025-01-15",
    "campaign_id": 101,
    "campaign_name": "US_Search_Mobile_1",
    "total_searches": 9658,       // proportionally distributed
    "monetized_searches": 7726,   // proportionally distributed
    "paid_clicks": 214,            // proportionally distributed
    "revenue": 55.02               // net (70%), 2 decimals
  },
  {
    "date": "2025-01-15",
    "campaign_id": 102,
    "campaign_name": "US_Search_Desktop_1",
    "total_searches": 5762,
    "monetized_searches": 4610,
    "paid_clicks": 128,
    "revenue": 32.80
  }
]
```

**Requirements:**
- Sort results by date, then by campaign_id
- Date queries are inclusive (from <= date <= to)
- Return empty array [] for no data (not an error)

**Authentication:**
- API key in query parameter `key` (like real SearchBarista)
- Each key maps to one traffic_source_id
- Validate that `ts` parameter matches the key's traffic_source_id
- Return 401 for invalid key, 403 for mismatched source

### 4. Testing (30 min)

Provide evidence that your solution works:

```bash
# 1. Process the data
python process_data.py

# 2. Test API endpoint
# For Supabase:
# BASE_URL="http://localhost:54321/functions/v1/pubstats"
# For FastAPI/Other:
# BASE_URL="http://localhost:8000/pubstats"

curl "${BASE_URL}?ts=66&from=2025-01-15&to=2025-01-20&key=test_key_66"

# 3. Verify idempotency (run again, same results)
python process_data.py
```

## Deliverables

### For Supabase Implementation:
```
your-solution/
├── README.md              # Setup instructions
├── schema.sql            # Database migrations
├── process_data.py       # Processing script
├── supabase/functions/
│   └── pubstats/
│       └── index.ts      # API endpoint
├── test_results.json     # Your actual API outputs
├── calculation_explanation.md  # MANUAL explanation of calculations
├── edge_cases_handled.md      # How you handled edge cases
├── ai_usage_log.md            # How you used AI (prompts, corrections)
└── time_log.txt              # Time spent on each part
```

### For Non-Supabase Implementation:
```
your-solution/
├── README.md              # Setup instructions
├── Dockerfile            # Required for deployment
├── docker-compose.yml    # Database + app setup
├── Makefile              # Commands: make run, make ingest, make test
├── schema.sql            # Database migrations
├── process_data.py       # Processing script
├── app.py                # API server (FastAPI/Flask/etc)
├── requirements.txt      # Python dependencies
├── test_results.json     # Your actual API outputs
├── calculation_explanation.md  # MANUAL explanation of calculations
├── edge_cases_handled.md      # How you handled edge cases
├── ai_usage_log.md            # How you used AI (prompts, corrections)
└── time_log.txt              # Time spent on each part
```

## Evaluation Criteria

Remember: your implementation must reflect the publisher perspective. Providers are data sources only; publishers consume the API results.

### Must Have (Pass/Fail)
- [ ] Revenue distribution is proportional to clicks
- [ ] Commission applied correctly (70% to publisher)
- [ ] API returns correct format as per API_EXAMPLES.md
- [ ] Authentication works (key validation + source matching)
- [ ] Idempotent processing (can run multiple times)
- [ ] For non-Supabase: Docker setup with `make run` command

### Critical Requirements (Instant Fail if Missing)
- [ ] Manual calculation shown in calculation_explanation.md
- [ ] All edge cases from DATA_NOTES.md documented in edge_cases_handled.md
- [ ] AI usage transparently documented with specific prompts
- [ ] At least one example of correcting AI's mistake
- [ ] Evidence of running process_data.py twice (idempotency check)

### Quality Checks
- [ ] Clean, readable code (not just AI-generated)
- [ ] Proper error handling for edge cases
- [ ] Evidence of understanding the business logic
- [ ] Thoughtful handling of data anomalies

### Optional Bonus Features (not required)
- [ ] CSV export format (?format=csv parameter)
- [ ] Company name resolution for traffic sources
- [ ] Detailed logging with structured output
- [ ] Performance optimization for large datasets

## Provided Files

| File | Description |
|------|-------------|
| test_searchbarista_feed_data.csv | Feed revenue data |
| test_binom_stats_clicks.csv | Campaign click data |
| DATA_MAPPING.md | Complete traffic_source → company mapping |
| DISTRIBUTION_ALGORITHM.md | Concept & constraints to formalize |
| VALIDATION_DATA.csv | Sample data to verify your calculations |

## Important Notes

1. **Matching Keys:** Feed_id from feed CSV must exactly match tracking_tag from clicks CSV (case-sensitive)
2. **Commission:** Publishers receive 70% (we keep 30%)
3. **Precision:**
   - Revenue: Store at 4 decimals in DB, return 2 decimals in API
   - Integer metrics: Use "largest remainder" method for reconciliation
   - All metrics distributed by same click proportion
4. **No clicks:** If no clicks for a feed/date, skip (no allocation)
5. **Test keys:** Use test_key_66, test_key_67, test_key_68 for testing
6. **Security:** Never log API keys or full URLs with keys in console/logs
7. **Idempotency:** Check file SHA256 hash before processing to prevent duplicates
8. **Internal metrics:** Clicks are for calculation only - NEVER expose in API response

## Required Documentation

### 1. calculation_explanation.md
Provide MANUAL step-by-step calculation for at least 3 examples:
- Show the exact formula you used
- Calculate one example BY HAND (show all steps)
- Explain why campaign 101 gets X amount on 2025-01-15

### 2. edge_cases_handled.md
Document how you handled:
- Duplicate rows in CSV
- Zero or negative clicks
- Case sensitivity in Feed_id/tracking_tag
- Revenue with no clicks
- Very small amounts (rounding)

### 3. ai_usage_log.md
Be transparent about AI usage:
- What prompts did you use?
- What did AI generate vs what you wrote?
- What errors did AI make that you fixed?
- Include at least one example where you corrected AI's output

## Time Budget

- Schema & setup: 30 min
- Processing script: 2 hours
- API endpoint: 1.5 hours
- Testing & validation: 30 min
- Required documentation: 1.5 hours
  - calculation_explanation.md: 30 min
  - edge_cases_handled.md: 30 min
  - ai_usage_log.md: 30 min
- Buffer: 1 hour

Total: 6.5 hours

---

Start with DISTRIBUTION_ALGORITHM.md to understand the concept & constraints.
Use VALIDATION_DATA.csv to check your calculation approach.