# Triple Whale Bridge - GHL to Triple Whale Transformation Skill

Use this skill when transforming GoHighLevel (GHL) CRM webhook data into Triple Whale API format for the `/data-in/orders` or `/data-in/event` endpoints.

## When to Use This Skill

- User asks to send GHL data to Triple Whale
- User wants to transform CRM pipeline data for attribution
- User needs to format webhook payloads for Triple Whale API
- User asks about GHL → Triple Whale integration

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

## Transformation Rules

### Pipeline Stage → Triple Whale Event Type

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

### Triple Whale Event Payload Format

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

### Triple Whale Orders Payload Format

For transaction/order data, use this format:

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

## Transformation Code Example

```python
def transform_ghl_to_triple_whale_event(ghl_payload: dict) -> dict:
    """Transform GHL webhook to Triple Whale event format."""

    # Map pipeline stages to event types
    stage_mapping = {
        "new lead": "lead",
        "contacted": "lead",
        "qualified": "mql",
        "marketing qualified": "mql",
        "sales qualified": "sql",
        "discovery complete": "sql",
        "demo scheduled": "book_demo",
        "proposal sent": "opportunity",
        "negotiation": "opportunity",
        "closed won": "custom",
        "closed lost": "custom",
    }

    stage = ghl_payload.get("pipelineStage", "").lower()
    event_type = stage_mapping.get(stage, "custom")

    # Calculate days in pipeline
    days_in_pipeline = None
    if ghl_payload.get("dateAdded"):
        from datetime import datetime
        added = datetime.fromisoformat(ghl_payload["dateAdded"].replace("Z", "+00:00"))
        days_in_pipeline = (datetime.now(added.tzinfo) - added).days

    # Build Triple Whale event
    tw_event = {
        "type": event_type,
        "email": ghl_payload.get("email", "").lower().strip() or None,
        "phone": ghl_payload.get("phone"),
        "timestamp": ghl_payload.get("dateUpdated") or datetime.utcnow().isoformat() + "Z",
        "properties": {
            "pipeline_name": ghl_payload.get("pipelineName"),
            "pipeline_stage": ghl_payload.get("pipelineStage"),
            "opportunity_name": ghl_payload.get("opportunityName"),
            "opportunity_id": ghl_payload.get("opportunityId"),
            "lead_value": ghl_payload.get("monetaryValue") or ghl_payload.get("leadValue"),
            "currency": "USD",
            "source": ghl_payload.get("attributionSource") or ghl_payload.get("source"),
            "ghl_contact_id": ghl_payload.get("contactId"),
            "ghl_opportunity_id": ghl_payload.get("opportunityId"),
            "company_name": ghl_payload.get("companyName"),
            "assigned_to": ghl_payload.get("assignedTo"),
        }
    }

    if days_in_pipeline is not None:
        tw_event["properties"]["days_in_pipeline"] = days_in_pipeline

    # Add custom event name for closed stages
    if stage == "closed won":
        tw_event["properties"]["event_name"] = "closed_won"
        tw_event["properties"]["value"] = ghl_payload.get("monetaryValue") or ghl_payload.get("leadValue")
    elif stage == "closed lost":
        tw_event["properties"]["event_name"] = "closed_lost"

    # Remove None values from properties
    tw_event["properties"] = {k: v for k, v in tw_event["properties"].items() if v is not None}

    return tw_event


def transform_ghl_to_triple_whale_order(ghl_payload: dict, shop_domain: str) -> dict:
    """Transform GHL closed-won deal to Triple Whale order format."""

    return {
        "shop": shop_domain,
        "order_id": ghl_payload.get("opportunityId") or f"GHL-{ghl_payload.get('contactId')}",
        "created_at": ghl_payload.get("dateAdded"),
        "updated_at": ghl_payload.get("dateUpdated"),
        "platform": "CUSTOM",
        "platform_account_id": ghl_payload.get("locationId"),
        "customer": {
            "email": ghl_payload.get("email", "").lower().strip(),
            "phone": ghl_payload.get("phone"),
            "first_name": ghl_payload.get("firstName"),
            "last_name": ghl_payload.get("lastName"),
        },
        "line_items": [
            {
                "product_id": ghl_payload.get("pipelineId", "service"),
                "variant_id": ghl_payload.get("pipelineStageId", "default"),
                "title": ghl_payload.get("opportunityName", "Service"),
                "quantity": 1,
                "price": ghl_payload.get("monetaryValue") or ghl_payload.get("leadValue") or 0,
            }
        ],
        "total_price": ghl_payload.get("monetaryValue") or ghl_payload.get("leadValue") or 0,
        "subtotal_price": ghl_payload.get("monetaryValue") or ghl_payload.get("leadValue") or 0,
        "total_tax": 0,
        "total_discounts": 0,
        "currency": "USD",
        "tags": ghl_payload.get("tags", []),
        "source_name": "gohighlevel",
    }
```

## Using the Triple Whale Bridge Service

The `triple_whale_bridge` service in this repository handles transformation automatically:

### Start the Bridge
```bash
cd triple_whale_bridge
export TRIPLE_WHALE_API_KEY="your_api_key"
python -m triple_whale_bridge --port 8000
```

### Configure GHL Webhook
Point GHL workflow webhook to: `https://your-domain.com/webhook/ghl`

### Test Transformation
```bash
curl -X POST http://localhost:8000/test/transform \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "pipelineName": "Main Sales Pipeline",
    "pipelineStage": "Qualified",
    "leadValue": 10000
  }'
```

## Required Fields Checklist

### For Events Endpoint
- [x] `type` - Event type (lead, mql, sql, book_demo, opportunity, custom)
- [x] `email` OR `phone` - At least one customer identifier
- [ ] `timestamp` - ISO 8601 format (optional, defaults to now)
- [ ] `properties` - Custom properties (optional)

### For Orders Endpoint
- [x] `shop` - Your store domain
- [x] `order_id` - Unique order identifier
- [x] `created_at` - Order creation timestamp
- [x] `platform` - Use "CUSTOM" for GHL
- [x] `customer.email` OR `customer.phone` - Customer identifier
- [x] `line_items` - Array of products/services
- [x] `total_price` - Total order amount

## Common Transformations

### Phone Number Normalization
```python
import re

def normalize_phone(phone: str) -> str:
    """Normalize to E.164 format."""
    if not phone:
        return None
    digits = re.sub(r"[^\d+]", "", phone)
    if len(digits) == 10:
        return f"+1{digits}"
    if len(digits) == 11 and digits.startswith("1"):
        return f"+{digits}"
    if not digits.startswith("+"):
        return f"+{digits}"
    return digits
```

### Email Normalization
```python
def normalize_email(email: str) -> str:
    """Normalize email for matching."""
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

For attribution to work:
1. Triple Pixel must be installed on your website
2. Pixel event fires BEFORE API event
3. Email/phone must match between Pixel and API
4. Events process in ~5 minutes (recent) or ~20 minutes (historical)
