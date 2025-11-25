# Triple Whale Bridge - Reference Code

## Complete Transformation Functions

### GHL to Triple Whale Event

```python
from datetime import datetime
import re


def transform_ghl_to_triple_whale_event(ghl_payload: dict) -> dict:
    """
    Transform GoHighLevel webhook payload to Triple Whale event format.

    Args:
        ghl_payload: Raw GHL webhook payload dict

    Returns:
        Triple Whale event dict ready for /data-in/event endpoint
    """

    # Pipeline stage to event type mapping
    stage_mapping = {
        "new lead": "lead",
        "contacted": "lead",
        "qualified": "mql",
        "marketing qualified": "mql",
        "sales qualified": "sql",
        "discovery complete": "sql",
        "demo scheduled": "book_demo",
        "discovery call": "book_demo",
        "proposal sent": "opportunity",
        "negotiation": "opportunity",
        "contract sent": "opportunity",
        "closed won": "custom",
        "closed lost": "custom",
    }

    stage = ghl_payload.get("pipelineStage", "").lower()
    event_type = stage_mapping.get(stage, "custom")

    # Calculate days in pipeline
    days_in_pipeline = None
    if ghl_payload.get("dateAdded"):
        try:
            added = datetime.fromisoformat(
                ghl_payload["dateAdded"].replace("Z", "+00:00")
            )
            days_in_pipeline = (datetime.now(added.tzinfo) - added).days
        except (ValueError, TypeError):
            pass

    # Normalize identifiers
    email = normalize_email(ghl_payload.get("email"))
    phone = normalize_phone(ghl_payload.get("phone"))

    # Build Triple Whale event
    tw_event = {
        "type": event_type,
        "email": email,
        "phone": phone,
        "timestamp": (
            ghl_payload.get("dateUpdated") or
            datetime.utcnow().isoformat() + "Z"
        ),
        "properties": {
            "pipeline_name": ghl_payload.get("pipelineName"),
            "pipeline_stage": ghl_payload.get("pipelineStage"),
            "opportunity_name": ghl_payload.get("opportunityName"),
            "opportunity_id": ghl_payload.get("opportunityId"),
            "lead_value": (
                ghl_payload.get("monetaryValue") or
                ghl_payload.get("leadValue")
            ),
            "currency": "USD",
            "source": (
                ghl_payload.get("attributionSource") or
                ghl_payload.get("source")
            ),
            "ghl_contact_id": ghl_payload.get("contactId"),
            "ghl_opportunity_id": ghl_payload.get("opportunityId"),
            "company_name": ghl_payload.get("companyName"),
            "assigned_to": ghl_payload.get("assignedTo"),
        }
    }

    if days_in_pipeline is not None:
        tw_event["properties"]["days_in_pipeline"] = days_in_pipeline

    # Handle closed stages
    if stage == "closed won":
        tw_event["properties"]["event_name"] = "closed_won"
        tw_event["properties"]["value"] = (
            ghl_payload.get("monetaryValue") or
            ghl_payload.get("leadValue")
        )
    elif stage == "closed lost":
        tw_event["properties"]["event_name"] = "closed_lost"

    # Remove None values from properties
    tw_event["properties"] = {
        k: v for k, v in tw_event["properties"].items()
        if v is not None
    }

    return tw_event


def transform_ghl_to_triple_whale_order(
    ghl_payload: dict,
    shop_domain: str
) -> dict:
    """
    Transform GHL closed-won deal to Triple Whale order format.

    Args:
        ghl_payload: Raw GHL webhook payload dict
        shop_domain: Your store domain for Triple Whale

    Returns:
        Triple Whale order dict ready for /data-in/orders endpoint
    """

    return {
        "shop": shop_domain,
        "order_id": (
            ghl_payload.get("opportunityId") or
            f"GHL-{ghl_payload.get('contactId')}"
        ),
        "created_at": ghl_payload.get("dateAdded"),
        "updated_at": ghl_payload.get("dateUpdated"),
        "platform": "CUSTOM",
        "platform_account_id": ghl_payload.get("locationId"),
        "customer": {
            "email": normalize_email(ghl_payload.get("email")),
            "phone": normalize_phone(ghl_payload.get("phone")),
            "first_name": ghl_payload.get("firstName"),
            "last_name": ghl_payload.get("lastName"),
        },
        "line_items": [
            {
                "product_id": ghl_payload.get("pipelineId", "service"),
                "variant_id": ghl_payload.get("pipelineStageId", "default"),
                "title": ghl_payload.get("opportunityName", "Service"),
                "quantity": 1,
                "price": (
                    ghl_payload.get("monetaryValue") or
                    ghl_payload.get("leadValue") or 0
                ),
            }
        ],
        "total_price": (
            ghl_payload.get("monetaryValue") or
            ghl_payload.get("leadValue") or 0
        ),
        "subtotal_price": (
            ghl_payload.get("monetaryValue") or
            ghl_payload.get("leadValue") or 0
        ),
        "total_tax": 0,
        "total_discounts": 0,
        "currency": "USD",
        "tags": ghl_payload.get("tags", []),
        "source_name": "gohighlevel",
    }


def normalize_phone(phone: str) -> str:
    """Normalize phone number to E.164 format."""
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


def normalize_email(email: str) -> str:
    """Normalize email address for matching."""
    return email.lower().strip() if email else None
```

## API Client with Retry

```python
import httpx
import asyncio
from typing import Optional


class TripleWhaleClient:
    """Async client for Triple Whale Data-In API."""

    BASE_URL = "https://api.triplewhale.com/api/v2"

    def __init__(self, api_key: str):
        self.api_key = api_key
        self.client = httpx.AsyncClient(
            base_url=self.BASE_URL,
            headers={
                "x-api-key": api_key,
                "Content-Type": "application/json",
            },
            timeout=30.0,
        )

    async def send_event(self, event: dict) -> dict:
        """Send event to /data-in/event with retry logic."""
        return await self._request_with_retry("POST", "/data-in/event", event)

    async def send_order(self, order: dict) -> dict:
        """Send order to /data-in/orders with retry logic."""
        return await self._request_with_retry("POST", "/data-in/orders", order)

    async def _request_with_retry(
        self,
        method: str,
        endpoint: str,
        json: dict,
        max_retries: int = 4,
    ) -> dict:
        """Make request with exponential backoff retry."""

        delays = [2, 4, 8, 16]  # seconds

        for attempt in range(max_retries + 1):
            try:
                response = await self.client.request(
                    method=method,
                    url=endpoint,
                    json=json,
                )

                if response.status_code == 200:
                    return response.json()

                # Don't retry auth/validation errors
                if response.status_code in (401, 403, 422):
                    raise Exception(f"API error {response.status_code}: {response.text}")

                # Retry on rate limit or server errors
                if response.status_code in (429, 500, 502, 503, 504):
                    if attempt < max_retries:
                        await asyncio.sleep(delays[attempt])
                        continue

                raise Exception(f"API error {response.status_code}: {response.text}")

            except httpx.RequestError as e:
                if attempt < max_retries:
                    await asyncio.sleep(delays[attempt])
                    continue
                raise

        raise Exception("Max retries exceeded")

    async def close(self):
        await self.client.aclose()
```

## Usage Example

```python
import asyncio


async def main():
    # Sample GHL webhook payload
    ghl_payload = {
        "email": "john@example.com",
        "phone": "(555) 123-4567",
        "firstName": "John",
        "lastName": "Doe",
        "companyName": "Acme Corp",
        "pipelineName": "Main Sales Pipeline",
        "pipelineStage": "Qualified",
        "opportunityId": "opp_123",
        "opportunityName": "Enterprise Deal",
        "leadValue": 10000,
        "dateAdded": "2024-01-01T10:00:00Z",
        "dateUpdated": "2024-01-15T14:30:00Z",
        "source": "facebook",
    }

    # Transform to Triple Whale event
    tw_event = transform_ghl_to_triple_whale_event(ghl_payload)
    print("Event payload:", tw_event)

    # Send to Triple Whale
    client = TripleWhaleClient(api_key="your_api_key")
    try:
        result = await client.send_event(tw_event)
        print("Success:", result)
    finally:
        await client.close()


if __name__ == "__main__":
    asyncio.run(main())
```

## Using the Bridge Service

The `triple_whale_bridge` service handles all transformations automatically:

```bash
# Start the bridge
cd triple_whale_bridge
export TRIPLE_WHALE_API_KEY="your_api_key"
python -m triple_whale_bridge --port 8000

# Test transformation
curl -X POST http://localhost:8000/test/transform \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "pipelineName": "Main Sales Pipeline",
    "pipelineStage": "Qualified",
    "leadValue": 10000
  }'

# Configure GHL webhook to point to:
# https://your-domain.com/webhook/ghl
```
