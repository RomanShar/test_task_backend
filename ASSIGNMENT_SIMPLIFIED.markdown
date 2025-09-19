# Backend Developer Test Assignment

**Time**: 8-10 hours | **Deadline**: 48 hours

## Business Context

SearchBarista is an intermediary platform for monetizing search traffic, primarily in English-speaking markets (e.g., USA). Publishers generate traffic from sources like Facebook or native ads and redirect it to specialized links (feeds) provided by feed providers (e.g., N2S for Yahoo/Bing/Google). Feed providers monetize this traffic via ads in search results.

- **Business Flow**:
  1. Publishers send traffic to feeds using campaign-specific feed IDs.
  2. Feed providers supply aggregated daily stats per feed_id (total searches, monetized searches, paid clicks, revenue).
  3. SearchBarista distributes these metrics proportionally to publisher campaigns based on clicks (clicks / total_feed_clicks).
  4. SearchBarista retains a 30% commission; publishers receive 70% net revenue.
  5. Publishers access their distributed stats via an API, never seeing raw feed data.

- **Key Metrics** (distributed proportionally):
  - **Total Searches**: User queries sent to the search engine.
  - **Monetized Searches**: Queries that displayed ads.
  - **Paid Clicks**: Clicks on paid ads.
  - **Revenue**: Gross revenue from clicks, net revenue after commission.

This task simulates processing feed and click data, distributing metrics, and exposing publisher-facing API results.

## Task Overview

Build a data pipeline that:
1. Processes two CSV files (feed revenue + campaign clicks)
2. Distributes revenue and metrics proportionally by clicks
3. Serves aggregated data via a simple HTTP API

## Core Requirements (Must-Have)

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
    feed_id VARCHAR(10) NOT NULL,
    clicks INTEGER,
    traffic_source_id INTEGER,
    campaign_name VARCHAR(100),
    PRIMARY KEY (date, campaign_id)
);

-- Calculated distributions
CREATE TABLE revenue_allocations (
    date DATE NOT NULL,
    campaign_id INTEGER NOT NULL,
    feed_id VARCHAR(10) NOT NULL,
    traffic_source_id INTEGER,
    campaign_name VARCHAR(100),
    clicks INTEGER,                     -- Internal, never exposed in API
    total_searches INTEGER,            -- Distributed
    monetized_searches INTEGER,        -- Distributed
    paid_clicks INTEGER,               -- Distributed ad clicks
    net_revenue DECIMAL(10,4),         -- After 30% commission
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

-- Indexes
CREATE INDEX idx_allocations_source ON revenue_allocations(traffic_source_id, date);
CREATE INDEX idx_allocations_date ON revenue_allocations(date);
CREATE INDEX idx_feed_data_date ON feed_data(date, feed_id);
CREATE INDEX idx_clicks_feed_id ON campaign_clicks(date, feed_id);
```

### 2. Data Processing Script (2.5 hours)

Create `process_data.py` that:
1. Loads `test_searchbarista_feed_data.csv` and `test_binom_stats_clicks.csv`.
2. For each `(date, feed_id)` from `feed_data`:
   - Finds matching campaigns in `campaign_clicks` where `feed_id` matches (case-sensitive).
   - Calculates click proportions: `clicks / total_feed_clicks`, where `total_feed_clicks` is the sum of `clicks` for all campaigns with matching `feed_id`.
   - Distributes metrics proportionally: `total_searches`, `monetized_searches`, `paid_clicks`, `revenue`.
   - Applies 30% commission to revenue (publishers get 70%).
   - Uses largest remainder method for integer metrics (`total_searches`, `monetized_searches`, `paid_clicks`).
   - Stores results in `revenue_allocations`.
3. Ensures idempotency by checking `file_sha256` in `file_ingestions` before processing.
4. Handles edge cases (see `DATA_NOTES.md`): duplicates, negative/zero clicks, case mismatches.

**Key test case**:
- Feed SB100 on 2025-01-15: $125.45 revenue, distribute to campaigns 101 and 102 based on clicks.

### 3. HTTP API Endpoint (1.5 hours)

Create an HTTP endpoint using **Supabase Edge Functions** or **FastAPI/Flask/Express**:

**Request**:
```
GET /pubstats?ts=66&from=2025-01-15&to=2025-01-20&key=test_key_66
```

**Response** (JSON, sorted by date, then campaign_id):
```json
[
  {
    "date": "2025-01-15",
    "campaign_id": 101,
    "campaign_name": "US_Search_Mobile_1",
    "total_searches": 9658,
    "monetized_searches": 7726,
    "paid_clicks": 214,
    "revenue": 55.02
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

**Requirements**:
- Authenticate via API key (`key` parameter). Validate `ts` matches key’s traffic_source_id.
- Return 401 for invalid key, 403 for mismatched source, 200 with `[]` for no data.
- Dates are inclusive (`from <= date <= to`).
- Revenue: Store at 4 decimals in DB, return 2 decimals in API.
- Use banker’s rounding for revenue, largest remainder for integer metrics.

### 4. Testing and Validation (1 hour)

1. Run `process_data.py` and verify results match `VALIDATION_DATA.csv` for provided cases.
2. Test API with:
```bash
# For Supabase:
BASE_URL="http://localhost:54321/functions/v1/pubstats"
# For FastAPI/Other:
BASE_URL="http://localhost:8000/pubstats"

curl "${BASE_URL}?ts=66&from=2025-01-15&to=2025-01-20&key=test_key_66"
```
3. Verify idempotency: Run `process_data.py` twice, ensure same results.
4. Use `VALIDATION_DATA.csv` for partial validation and compute full output for the API query above to ensure correctness.

### 5. Documentation (2-2.5 hours)

Provide:
- `calculation_explanation.md`: Manual calculations for 1-2 examples (e.g., SB100 on 2025-01-15). Show formula and steps.
- `edge_cases_handled.md`: How you handled duplicates, negative/zero clicks, case mismatches.
- `ai_usage_log.md` (only if AI used): List prompts, AI outputs, and at least one correction you made.
- `time_log.txt`: Hours spent on each part.

## Deliverables

### For Supabase:
```
your-solution/
├── README.md              # Setup instructions
├── schema.sql            # Database migrations
├── process_data.py       # Processing script
├── supabase/functions/
│   └── pubstats/
│       └── index.ts      # API endpoint
├── test_results.json     # API output for ts=66, 2025-01-15 to 2025-01-20
├── calculation_explanation.md
├── edge_cases_handled.md
├── ai_usage_log.md       # If AI used
└── time_log.txt
```

### For Non-Supabase:
```
your-solution/
├── README.md              # Setup instructions
├── Dockerfile            # Deployment
├── docker-compose.yml    # DB + app
├── Makefile              # make run, make ingest, make test
├── schema.sql
├── process_data.py
├── app.py                # API server
├── requirements.txt
├── test_results.json
├── calculation_explanation.md
├── edge_cases_handled.md
├── ai_usage_log.md       # If AI used
└── time_log.txt
```

## Evaluation Criteria

**Scoring Rubric** (100 points):
- Correct distribution (40): Proportional allocation, 30% commission, largest remainder for integers.
- API functionality (20): Correct response format, authentication, sorting.
- Documentation (20): Clear calculations, edge case handling, AI usage (if applicable).
- Edge case handling (10): Proper handling of duplicates, negative/zero clicks, case mismatches.
- Code quality (10): Clean, readable, error-handled code.

**Instant Fail**:
- No manual calculations in `calculation_explanation.md`.
- Missing edge case documentation.
- No idempotency check.
- Incorrect distribution or commission logic.

## Provided Files
- `test_searchbarista_feed_data.csv`: Feed revenue data
- `test_binom_stats_clicks.csv`: Campaign click data
- `DATA_MAPPING.md`: Traffic source to company mapping
- `DISTRIBUTION_ALGORITHM.md`: Distribution logic
- `API_EXAMPLES.md`: API request/response examples
- `VALIDATION_DATA.csv`: Sample data for validation
- `DATA_NOTES.md`: Notes on edge cases

## Important Notes
1. **Matching**: `feed_id` in `campaign_clicks` matches `feed_id` in `feed_data` (case-sensitive).
2. **Commission**: Publishers get 70% of revenue after distribution.
3. **Precision**: Revenue stored at 4 decimals, returned at 2 decimals. Integers use largest remainder.
4. **Edge Cases**: See `DATA_NOTES.md` for duplicates, negative/zero clicks.
5. **Test Keys**: Use `test_key_66`, `test_key_67`, `test_key_68`.
6. **Security**: Never log API keys.
7. **Idempotency**: Check `file_sha256` in `file_ingestions`.

## Time Budget
- Schema & setup: 30 min
- Processing script: 2.5 hours
- API endpoint: 1.5 hours
- Testing & validation: 1 hour
- Documentation: 2-2.5 hours
  - `calculation_explanation.md`: 45 min
  - `edge_cases_handled.md`: 45 min
  - `ai_usage_log.md`: 30 min (if applicable)
- Buffer: 1-2 hours

Start with `DISTRIBUTION_ALGORITHM.md` and validate against `VALIDATION_DATA.csv`.