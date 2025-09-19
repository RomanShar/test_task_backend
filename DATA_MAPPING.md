# Data Mapping Reference

## Key Relationships

### 1. Feed_id ↔ tracking_tag
**Direct match by string value**
- `Feed_id` from feed CSV = `tracking_tag` from clicks CSV
- Example: SB100 in feed matches SB100 in clicks

### 2. Traffic Source → Company Mapping

| traffic_source_id | Company Name | API Key |
|-------------------|--------------|---------|
| 66 | US Publisher Inc | test_key_66 |
| 67 | UK Media Ltd | test_key_67 |
| 68 | CA Traffic Corp | test_key_68 |
| 69 | DE Marketing GmbH | test_key_69 |
| 70 | AU Digital Pty | test_key_70 |
| 71 | NZ Ads Limited | test_key_71 |
| 72 | FR Publicité SA | test_key_72 |
| 73 | ES Medios SL | test_key_73 |
| 74 | IT Media SRL | test_key_74 |
| 75 | BR Digital Ltda | test_key_75 |

### 3. Campaign Ownership

| Campaign ID | Campaign Name | traffic_source_id | Feed (tracking_tag) |
|-------------|---------------|-------------------|---------------------|
| 101 | US_Search_Mobile_1 | 66 | SB100 |
| 102 | US_Search_Desktop_1 | 66 | SB100 |
| 103 | UK_Search_Mobile_1 | 67 | SB101 |
| 104 | CA_Search_All_1 | 68 | SB102 |
| 105 | CA_Search_Mobile_1 | 68 | SB102 |
| 106 | CA_Search_Desktop_1 | 68 | SB102 |
| 107 | AU_Search_All_1 | 70 | SB103 |
| 108 | NZ_Search_Mobile_1 | 71 | SB104 |
| 109 | DE_Search_All_1 | 69 | SB105 |
| 110 | FR_Search_Mobile_1 | 72 | SB106 |
| 111 | ES_Search_Desktop_1 | 73 | SB107 |
| 112 | IT_Search_Mobile_1 | 74 | SB108 |
| 113 | BR_Search_All_1 | 75 | SB109 |

## Distribution Rules

### Primary Rule: Feed → Campaigns → Company
1. Each Feed_id distributes revenue to campaigns with matching tracking_tag
2. Distribution proportion based on clicks: `campaign_clicks / total_feed_clicks`
3. Company gets sum of all its campaigns (identified by traffic_source_id)

### Example Distribution

**Date: 2025-01-15, Feed: SB100, Revenue: $125.45**

Campaigns with SB100:
- Campaign 101: 5,420 clicks (62.65%)
- Campaign 102: 3,231 clicks (37.35%)
- Total: 8,651 clicks

Distribution:
- Campaign 101: $125.45 × 0.6265 = $78.60
- Campaign 102: $125.45 × 0.3735 = $46.85

After 30% commission:
- Campaign 101: $78.60 × 0.70 = $55.02 → Company 66
- Campaign 102: $46.85 × 0.70 = $32.80 → Company 66
- **Company 66 total for this feed/date: $87.82**