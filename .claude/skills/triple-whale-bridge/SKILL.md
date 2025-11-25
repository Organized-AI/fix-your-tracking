---
name: triple-whale-bridge
description: Transform GoHighLevel CRM webhook data into Triple Whale API format for /data-in/orders or /data-in/event endpoints. Use when sending GHL pipeline data to Triple Whale for attribution.
---

# Triple Whale Bridge - GHL to Triple Whale Transformation

Transform GoHighLevel (GHL) CRM webhook data into Triple Whale API format for full-funnel marketing attribution.

## Triple Whale API Endpoints

### Events Endpoint (Recommended for CRM Attribution)
```
POST https://api.triplewhale.com/api/v2/data-in/event
```
**Rate Limit:** 1,000 events/min | **Max Payload:** 3 KB

### Orders Endpoint (For E-commerce/Transactions)
```
POST https://api.triplewhale.com/api/v2/data-in/orders
```
**Rate Limit:** 25,000 requests/min

## Authentication

```bash
curl -X POST https://api.triplewhale.com/api/v2/data-in/event \
  -H "x-api-key: YOUR_TRIPLE_WHALE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{...}'
```

## GHL Webhook Payload Structure

GoHighLevel sends webhooks with this structure for pipeline/opportunity events:

```json
{
  "type": "OpportunityStageUpdate",
  "locationId": "loc_xxx",
  "contactId": "contact_xxx",
  "email": "customer@example.com",
  "phone": "+15551234567",
  "firstName": "John",
  "lastName": "Doe",
  "fullName": "John Doe",
  "companyName": "Acme Corp",
  "tags": ["enterprise", "demo-requested"],
  "source": "facebook",
  "attributionSource": "facebook_ads",
  "opportunityId": "opp_xxx",
  "opportunityName": "Enterprise Deal",
  "pipelineId": "pipe_xxx",
  "pipelineName": "Main Sales Pipeline",
  "pipelineStage": "Qualified",
  "pipelineStageId": "stage_xxx",
  "status": "open",
  "leadValue": 10000,
  "monetaryValue": 10000,
  "assignedTo": "user_xxx",
  "dateAdded": "2024-01-01T10:00:00Z",
  "dateUpdated": "2024-01-15T14:30:00Z",
  "customFields": [
    {"key": "utm_source", "value": "facebook"},
    {"key": "utm_campaign", "value": "q1_promo"}
  ]
}
```

## Pipeline Stage → Triple Whale Event Type Mapping

| GHL Pipeline Stage | TW Event Type | Description |
|-------------------|---------------|-------------|
| New Lead | `lead` | Initial lead capture |
| Contacted | `lead` | Lead contacted |
| Qualified / Marketing Qualified | `mql` | Marketing qualified |
| Sales Qualified / Discovery Complete | `sql` | Sales qualified |
| Demo Scheduled / Discovery Call | `book_demo` | Meeting booked |
| Proposal Sent / Negotiation | `opportunity` | Active deal |
| Closed Won | `custom` | Won deal (include revenue) |
| Closed Lost | `custom` | Lost deal |

## Triple Whale Event Payload Format

```json
{
  "type": "mql",
  "email": "customer@example.com",
  "phone": "+15551234567",
  "timestamp": "2024-01-15T14:30:00Z",
  "properties": {
    "pipeline_name": "Main Sales Pipeline",
    "pipeline_stage": "Qualified",
    "opportunity_name": "Enterprise Deal",
    "opportunity_id": "opp_xxx",
    "lead_value": 10000,
    "value": 1000,
    "currency": "USD",
    "source": "facebook_ads",
    "ghl_contact_id": "contact_xxx",
    "ghl_opportunity_id": "opp_xxx",
    "company_name": "Acme Corp",
    "assigned_to": "user_xxx",
    "days_in_pipeline": 14,
    "event_name": "qualified_lead"
  }
}
```

## Triple Whale Orders Payload Format

For transaction/order data (closed won deals):

```json
{
  "shop": "yourstore.com",
  "order_id": "ORD-12345",
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z",
  "platform": "CUSTOM",
  "platform_account_id": "your_ghl_location_id",
  "customer": {
    "email": "customer@example.com",
    "phone": "+15551234567",
    "first_name": "John",
    "last_name": "Doe"
  },
  "line_items": [
    {
      "product_id": "service_001",
      "variant_id": "enterprise_plan",
      "title": "Enterprise Plan",
      "quantity": 1,
      "price": 10000
    }
  ],
  "total_price": 10000,
  "subtotal_price": 10000,
  "total_tax": 0,
  "total_discounts": 0,
  "currency": "USD",
  "tags": ["enterprise", "annual"],
  "source_name": "gohighlevel"
}
```

## Required Fields

### For Events Endpoint
- `type` - Event type (lead, mql, sql, book_demo, opportunity, custom)
- `email` OR `phone` - At least one customer identifier
- `timestamp` - ISO 8601 format (optional, defaults to now)
- `properties` - Custom properties (optional)

### For Orders Endpoint
- `shop` - Your store domain
- `order_id` - Unique order identifier
- `created_at` - Order creation timestamp
- `platform` - Use "CUSTOM" for GHL
- `customer.email` OR `customer.phone` - Customer identifier
- `line_items` - Array of products/services
- `total_price` - Total order amount

## Data Normalization

### Phone (E.164 Format)
```python
import re
def normalize_phone(phone: str) -> str:
    if not phone:
        return None
    digits = re.sub(r"[^\d+]", "", phone)
    if len(digits) == 10:
        return f"+1{digits}"
    if len(digits) == 11 and digits.startswith("1"):
        return f"+{digits}"
    return f"+{digits}" if not digits.startswith("+") else digits
```

### Email
```python
def normalize_email(email: str) -> str:
    return email.lower().strip() if email else None
```

## Error Handling

| HTTP Code | Meaning | Action |
|-----------|---------|--------|
| 200 | Success | Event accepted |
| 401 | Invalid API key | Check TRIPLE_WHALE_API_KEY |
| 403 | Missing scope | Enable "Orders: Write" scope |
| 422 | Validation error | Check required fields |
| 429 | Rate limit | Wait and retry with backoff |
| 5xx | Server error | Retry with exponential backoff |

## Attribution Requirements

1. Triple Pixel must be installed on your website
2. Pixel event fires BEFORE API event
3. Email/phone must match between Pixel and API
4. Events process in ~5 minutes (recent) or ~20 minutes (historical)
