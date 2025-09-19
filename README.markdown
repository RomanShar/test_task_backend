# Backend Developer Test Assignment

**Time**: 8-10 hours | **Deadline**: 48 hours

## 📋 Package Contents

| File | Description |
|------|-------------|
| **ASSIGNMENT_SIMPLIFIED.md** | Main requirements - START HERE! |
| **DISTRIBUTION_ALGORITHM.md** | Concept & constraints (formalize the math) |
| **DATA_MAPPING.md** | Traffic source → company mappings |
| **API_EXAMPLES.md** | API request/response examples |
| **test_searchbarista_feed_data.csv** | Feed revenue input data (contains edge cases) |
| **test_binom_stats_clicks.csv** | Campaign click data (contains anomalies) |
| **VALIDATION_DATA.csv** | Sample calculations (partial, includes full output for ts=66, 2025-01-15 to 2025-01-20) |
| **DATA_NOTES.md** | Important notes about data quality issues |

## 🚀 Quick Start

1. Read **ASSIGNMENT_SIMPLIFIED.md** for requirements
2. Study **DISTRIBUTION_ALGORITHM.md** for concept & constraints
3. Check **DATA_MAPPING.md** for data relationships
4. Use **API_EXAMPLES.md** for API structure
5. Calculate results and validate against **VALIDATION_DATA.csv** (partial, but includes full output for one API query)

## 🛠️ Implementation Options

You can implement the API endpoint using either:
- **Option A**: Supabase Edge Functions (recommended)
- **Option B**: FastAPI, Flask, Express, or any web framework

The API contract must remain identical regardless of your choice. For non-Supabase implementations, include a `Dockerfile` and `Makefile` for easy setup (see optional templates in `examples/` if provided).

## ✅ Success Criteria

Your solution MUST:
- Correctly distribute revenue and metrics proportionally by clicks
- Apply 30% commission (publishers get 70% net revenue)
- Return API responses in the format shown in API_EXAMPLES.md
- Be idempotent (can run multiple times safely)
- Handle all edge cases in the test data
- Include manual calculations proving you understand the logic
- Document your AI usage transparently

## 📊 Data Processing Goals

After processing both CSV files, you should:
- Distribute ALL feed metrics proportionally by clicks
  - Revenue, total_searches, monetized_searches, paid_clicks
- Apply 30% platform commission to revenue only (publishers get 70%)
- Store results in `revenue_allocations` table
- Serve distributed data via API (never expose raw clicks)

The feed provider supplies only aggregate revenue and metrics. Publishers never see feed-level data. The goal is to fairly allocate provider revenue across publishers' campaigns, proportional to their click contribution.

## 🔑 Test API Keys

Use these keys for testing your API endpoint:
```
test_key_66 → Traffic Source 66 (US Publisher Inc)
test_key_67 → Traffic Source 67 (UK Media Ltd)
test_key_68 → Traffic Source 68 (CA Traffic Corp)
```
Note: Only these three keys need to work in your implementation. Other traffic sources in DATA_MAPPING.md are for reference only.

## ⏱️ Suggested Time Budget

- Database setup: 30 min
- Data processing script: 2.5 hours
- API endpoint: 1.5 hours
- Testing & validation: 1 hour
- Required documentation: 2-2.5 hours
  - calculation_explanation.md: 45 min
  - edge_cases_handled.md: 45 min
  - ai_usage_log.md: 30 min
- **Total: 8-10 hours**

## 📝 What to Submit

1. GitHub repository with your solution
2. API endpoint URL (Supabase deployed URL or local BASE_URL with Docker/Make instructions for FastAPI/Other)
3. Brief README with setup instructions
4. Time log showing hours spent on each part

Good luck!