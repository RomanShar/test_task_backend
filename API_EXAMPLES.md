# API Examples & Testing Guide

## Authentication

### API Key Methods

#### Option 1: Query Parameter (Like Real SearchBarista)
```javascript
// Extract API key from query parameter
const url = new URL(req.url);
const apiKey = url.searchParams.get('key');

const API_KEYS = {
  'test_key_66': 66,
  'test_key_67': 67,
  'test_key_68': 68,
};

const trafficSourceId = API_KEYS[apiKey];
if (!trafficSourceId) {
  return new Response(JSON.stringify({ error: 'Invalid API key' }), {
    status: 401,
    headers: { 'Content-Type': 'application/json' }
  });
}
```

#### Option 2: Authorization Header (Alternative)
```javascript
// Extract API key from Authorization header
const apiKey = req.headers.get('Authorization')?.replace('Bearer ', '');
// ... rest of validation
```

## API Endpoint

### Path Options:
- **Supabase:** `GET /functions/v1/pubstats`
- **FastAPI/Other:** `GET /pubstats`

The path differs based on implementation, but parameters and response format remain identical.

### Example 1: Basic Request

**Request (Supabase):**
```bash
curl "https://your-project.supabase.co/functions/v1/pubstats?ts=66&from=2025-01-15&to=2025-01-15&key=test_key_66"
```

**Request (FastAPI/Other):**
```bash
curl "http://localhost:8000/pubstats?ts=66&from=2025-01-15&to=2025-01-15&key=test_key_66"
```

**Response Format (all values are after proportional distribution):**
```json
[
  {
    "date": "2025-01-15",
    "campaign_id": 101,
    "campaign_name": "US_Search_Mobile_1",
    "total_searches": 9658,      // proportionally distributed
    "monetized_searches": 7726,  // proportionally distributed
    "paid_clicks": 214,           // proportionally distributed
    "revenue": 55.02              // net (70%), 2 decimal places
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

### Example 2: Multi-Day Request

**Request (choose based on your implementation):**
```bash
# Supabase:
curl "https://your-project.supabase.co/functions/v1/pubstats?ts=68&from=2025-01-15&to=2025-01-17&key=test_key_68"

# FastAPI/Other:
curl "http://localhost:8000/pubstats?ts=68&from=2025-01-15&to=2025-01-17&key=test_key_68"
```

**Response Format:**
```json
[
  {
    "date": "2025-01-15",
    "campaign_id": 104,
    "campaign_name": "CA_Search_All_1",
    "total_searches": <distributed_value>,
    "monetized_searches": <distributed_value>,
    "paid_clicks": <distributed_value>,
    "revenue": <calculated_net_revenue>
  },
  {
    "date": "2025-01-15",
    "campaign_id": 105,
    "campaign_name": "CA_Search_Mobile_1",
    "total_searches": <distributed_value>,
    "monetized_searches": <distributed_value>,
    "paid_clicks": <distributed_value>,
    "revenue": <calculated_net_revenue>
  }
  // ... more records for the date range
]
```

### Example 3: Invalid API Key

**Request:**
```bash
curl "https://your-project.supabase.co/functions/v1/pubstats?ts=66&from=2025-01-15&to=2025-01-15&key=wrong_key"
```

**Expected Response:**
```json
{
  "error": "Invalid API key"
}
```
Status: 401

### Example 4: Mismatched Source and Key

**Request:**
```bash
curl "https://your-project.supabase.co/functions/v1/pubstats?ts=67&from=2025-01-15&to=2025-01-15&key=test_key_66"
```

**Expected Response:**
```json
{
  "error": "Access denied: API key does not match traffic source"
}
```
Status: 403

### Example 5: No Data Found

**Request:**
```bash
curl "https://your-project.supabase.co/functions/v1/pubstats?ts=66&from=2025-12-01&to=2025-12-31&key=test_key_66"
```

**Expected Response:**
```json
[]
```
Status: 200 (empty array)

## Error Codes

| Status | Meaning | Example Response |
|--------|---------|------------------|
| 200 | Success | Data returned |
| 400 | Bad Request | `{"error": "Missing required parameter: from"}` |
| 401 | Unauthorized | `{"error": "Invalid API key"}` |
| 403 | Forbidden | `{"error": "Access denied"}` |
| 500 | Server Error | `{"error": "Internal server error"}` |

## Testing Checklist

```bash
# Adjust the base URL based on your implementation:
# Supabase: https://your-project.supabase.co/functions/v1/pubstats
# FastAPI/Other: http://localhost:8000/pubstats
BASE_URL="YOUR_BASE_URL_HERE"

# 1. Test valid request
curl "${BASE_URL}?ts=66&from=2025-01-15&to=2025-01-20&key=test_key_66"

# 2. Test missing auth
curl "${BASE_URL}?ts=66&from=2025-01-15&to=2025-01-20"

# 3. Test wrong key
curl "${BASE_URL}?ts=66&from=2025-01-15&to=2025-01-20&key=invalid_key"

# 4. Test missing parameter
curl "${BASE_URL}?ts=66&key=test_key_66"

# 5. Test cross-source access (should fail)
curl "${BASE_URL}?ts=67&from=2025-01-15&to=2025-01-20&key=test_key_66"

# 6. Test CSV format (BONUS - not required)
# curl "${BASE_URL}?ts=66&from=2025-01-15&to=2025-01-20&key=test_key_66&format=csv"
```

## Response Structure Requirements

Your API must return an array of objects with these fields:

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| date | string | Date in YYYY-MM-DD format | Yes |
| campaign_id | integer | Campaign identifier | Yes |
| campaign_name | string | Campaign name | Yes |
| total_searches | integer | Total searches (proportionally distributed) | Yes |
| monetized_searches | integer | Monetized searches (proportionally distributed) | Yes |
| paid_clicks | integer | Ad clicks from feed (proportionally distributed) | Yes |
| revenue | float | Net revenue (70% after commission), 2 decimals | Yes |

Note: `paid_clicks` in response is the distributed feed metric. The `campaign_clicks` field is internal and never returned in API.

**Important:**
- Revenue shown is what the publisher receives (70% of gross)
- Sort by date, then by campaign_id
- API returns 2 decimal places by default (add ?precision=4 for 4 decimals)
- CSV format with ?format=csv is BONUS (not required)
  - CSV returns the same fields in the same order as JSON
- Empty array `[]` for no data (not an error)
- Dates in queries are inclusive (from <= date <= to)