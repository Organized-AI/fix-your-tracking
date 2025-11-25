# Transformation Code Reference

Complete code examples for transforming GoHighLevel webhooks to Triple Whale format.

## Python Implementation

### Complete Transformer

```python
"""
GHL to Triple Whale Transformer
Full implementation with all edge cases handled.
"""

from datetime import datetime
from typing import Any, Optional
import re


class GHLToTripleWhaleTransformer:
    """Transform GoHighLevel webhooks to Triple Whale events."""

    # Pipeline stage to event type mapping
    STAGE_MAPPING = {
        # Lead stages
        "new lead": "lead",
        "new": "lead",
        "inbound": "lead",
        "inquiry": "lead",
        "cold lead": "lead",
        "contacted": "lead",
        "reached out": "lead",

        # MQL stages
        "qualified": "mql",
        "mql": "mql",
        "marketing qualified": "mql",
        "engaged": "mql",
        "warm lead": "mql",
        "hot lead": "mql",

        # SQL stages
        "sales qualified": "sql",
        "sql": "sql",
        "discovery complete": "sql",
        "demo complete": "sql",
        "needs assessed": "sql",

        # Demo stages
        "demo scheduled": "book_demo",
        "demo booked": "book_demo",
        "discovery call": "book_demo",
        "meeting set": "book_demo",
        "consultation booked": "book_demo",

        # Opportunity stages
        "proposal": "opportunity",
        "proposal sent": "opportunity",
        "quote sent": "opportunity",
        "negotiation": "opportunity",
        "contract sent": "opportunity",
        "contract review": "opportunity",

        # Closed stages
        "closed won": "custom",
        "won": "custom",
        "customer": "custom",
        "closed lost": "custom",
        "lost": "custom",
    }

    # Value multipliers for weighted attribution
    VALUE_MULTIPLIERS = {
        "lead": 0.0,
        "mql": 0.10,
        "sql": 0.25,
        "book_demo": 0.20,
        "opportunity": 0.50,
        "custom": 1.0,
    }

    def transform_to_event(self, ghl_payload: dict) -> Optional[dict]:
        """
        Transform GHL webhook to Triple Whale event.

        Args:
            ghl_payload: Raw GHL webhook payload

        Returns:
            Triple Whale event dict or None if invalid
        """
        # Extract and normalize identifiers
        email = self.normalize_email(ghl_payload.get("email"))
        phone = self.normalize_phone(ghl_payload.get("phone"))

        # Must have at least one identifier
        if not email and not phone:
            return None

        # Determine event type from stage
        stage = ghl_payload.get("pipelineStage", "").lower().strip()
        event_type = self.STAGE_MAPPING.get(stage, "custom")

        # Get deal value
        deal_value = self._get_monetary_value(ghl_payload)

        # Calculate attributed value
        multiplier = self.VALUE_MULTIPLIERS.get(event_type, 1.0)
        attributed_value = deal_value * multiplier if deal_value else None

        # Calculate days in pipeline
        days_in_pipeline = self._calculate_days_in_pipeline(
            ghl_payload.get("dateAdded")
        )

        # Build event
        event = {
            "type": event_type,
            "email": email,
            "phone": phone,
            "timestamp": self._get_timestamp(ghl_payload),
            "properties": self._build_properties(
                ghl_payload,
                stage,
                attributed_value,
                days_in_pipeline
            ),
        }

        # Remove None values
        event = {k: v for k, v in event.items() if v is not None}
        event["properties"] = {
            k: v for k, v in event["properties"].items()
            if v is not None
        }

        return event

    def transform_to_order(
        self,
        ghl_payload: dict,
        shop_domain: str
    ) -> Optional[dict]:
        """
        Transform GHL closed-won to Triple Whale order.

        Args:
            ghl_payload: Raw GHL webhook payload
            shop_domain: Your store domain

        Returns:
            Triple Whale order dict or None if invalid
        """
        email = self.normalize_email(ghl_payload.get("email"))
        phone = self.normalize_phone(ghl_payload.get("phone"))

        if not email and not phone:
            return None

        deal_value = self._get_monetary_value(ghl_payload) or 0

        return {
            "shop": shop_domain,
            "order_id": (
                ghl_payload.get("opportunityId") or
                f"GHL-{ghl_payload.get('contactId', 'unknown')}"
            ),
            "created_at": ghl_payload.get("dateAdded"),
            "updated_at": ghl_payload.get("dateUpdated"),
            "platform": "CUSTOM",
            "platform_account_id": ghl_payload.get("locationId"),
            "customer": {
                "email": email,
                "phone": phone,
                "first_name": ghl_payload.get("firstName"),
                "last_name": ghl_payload.get("lastName"),
            },
            "line_items": [
                {
                    "product_id": ghl_payload.get("pipelineId", "service"),
                    "variant_id": ghl_payload.get("pipelineStageId", "default"),
                    "title": ghl_payload.get("opportunityName", "Service"),
                    "quantity": 1,
                    "price": deal_value,
                }
            ],
            "total_price": deal_value,
            "subtotal_price": deal_value,
            "total_tax": 0,
            "total_discounts": 0,
            "currency": "USD",
            "tags": ghl_payload.get("tags", []),
            "source_name": "gohighlevel",
        }

    def _build_properties(
        self,
        ghl_payload: dict,
        stage: str,
        attributed_value: Optional[float],
        days_in_pipeline: Optional[int],
    ) -> dict:
        """Build event properties from payload."""
        props = {
            "pipeline_name": ghl_payload.get("pipelineName"),
            "pipeline_stage": ghl_payload.get("pipelineStage"),
            "opportunity_name": ghl_payload.get("opportunityName"),
            "opportunity_id": ghl_payload.get("opportunityId"),
            "lead_value": self._get_monetary_value(ghl_payload),
            "value": attributed_value,
            "currency": "USD",
            "source": (
                ghl_payload.get("attributionSource") or
                ghl_payload.get("source")
            ),
            "ghl_contact_id": ghl_payload.get("contactId"),
            "ghl_opportunity_id": ghl_payload.get("opportunityId"),
            "company_name": ghl_payload.get("companyName"),
            "assigned_to": ghl_payload.get("assignedTo"),
            "days_in_pipeline": days_in_pipeline,
        }

        # Add custom event name for closed stages
        if "closed won" in stage or stage == "won":
            props["event_name"] = "closed_won"
            props["value"] = self._get_monetary_value(ghl_payload)
        elif "closed lost" in stage or stage == "lost":
            props["event_name"] = "closed_lost"
            props["value"] = 0

        return props

    def _get_monetary_value(self, ghl_payload: dict) -> Optional[float]:
        """Extract monetary value from payload."""
        value = (
            ghl_payload.get("monetaryValue") or
            ghl_payload.get("leadValue")
        )

        if value is None:
            return None

        if isinstance(value, (int, float)):
            return float(value)

        if isinstance(value, str):
            # Remove currency symbols and commas
            cleaned = re.sub(r"[^\d.]", "", value)
            try:
                return float(cleaned)
            except ValueError:
                return None

        return None

    def _get_timestamp(self, ghl_payload: dict) -> str:
        """Get timestamp from payload or generate current."""
        ts = (
            ghl_payload.get("dateUpdated") or
            ghl_payload.get("timestamp")
        )

        if ts:
            return ts if ts.endswith("Z") else f"{ts}Z"

        return datetime.utcnow().isoformat() + "Z"

    def _calculate_days_in_pipeline(
        self,
        date_added: Optional[str]
    ) -> Optional[int]:
        """Calculate days since opportunity creation."""
        if not date_added:
            return None

        try:
            added = datetime.fromisoformat(
                date_added.replace("Z", "+00:00")
            )
            now = datetime.now(added.tzinfo)
            return (now - added).days
        except (ValueError, TypeError):
            return None

    @staticmethod
    def normalize_email(email: Optional[str]) -> Optional[str]:
        """Normalize email address."""
        if not email:
            return None
        return email.lower().strip()

    @staticmethod
    def normalize_phone(phone: Optional[str]) -> Optional[str]:
        """Normalize phone to E.164 format."""
        if not phone:
            return None

        # Remove all non-digit characters except +
        digits = re.sub(r"[^\d+]", "", phone)

        # Handle various formats
        if digits.startswith("+"):
            return digits
        elif len(digits) == 10:
            return f"+1{digits}"
        elif len(digits) == 11 and digits.startswith("1"):
            return f"+{digits}"
        elif len(digits) > 10:
            return f"+{digits}"

        return phone
```

---

## JavaScript/TypeScript Implementation

```typescript
/**
 * GHL to Triple Whale Transformer
 */

interface GHLPayload {
  email?: string;
  phone?: string;
  firstName?: string;
  lastName?: string;
  companyName?: string;
  pipelineName?: string;
  pipelineStage?: string;
  opportunityId?: string;
  opportunityName?: string;
  contactId?: string;
  locationId?: string;
  leadValue?: number | string;
  monetaryValue?: number | string;
  source?: string;
  attributionSource?: string;
  assignedTo?: string;
  dateAdded?: string;
  dateUpdated?: string;
  tags?: string[];
}

interface TripleWhaleEvent {
  type: string;
  email?: string;
  phone?: string;
  timestamp: string;
  properties: Record<string, unknown>;
}

const STAGE_MAPPING: Record<string, string> = {
  "new lead": "lead",
  "qualified": "mql",
  "sales qualified": "sql",
  "demo scheduled": "book_demo",
  "proposal sent": "opportunity",
  "closed won": "custom",
  "closed lost": "custom",
};

function transformGHLToTripleWhale(
  payload: GHLPayload
): TripleWhaleEvent | null {
  const email = payload.email?.toLowerCase().trim() || null;
  const phone = normalizePhone(payload.phone);

  if (!email && !phone) {
    return null;
  }

  const stage = payload.pipelineStage?.toLowerCase() || "";
  const eventType = STAGE_MAPPING[stage] || "custom";
  const dealValue = getMonetaryValue(payload);

  const event: TripleWhaleEvent = {
    type: eventType,
    email: email || undefined,
    phone: phone || undefined,
    timestamp: payload.dateUpdated || new Date().toISOString(),
    properties: {
      pipeline_name: payload.pipelineName,
      pipeline_stage: payload.pipelineStage,
      opportunity_name: payload.opportunityName,
      opportunity_id: payload.opportunityId,
      lead_value: dealValue,
      currency: "USD",
      source: payload.attributionSource || payload.source,
      ghl_contact_id: payload.contactId,
      company_name: payload.companyName,
    },
  };

  // Handle closed stages
  if (stage.includes("closed won") || stage === "won") {
    event.properties.event_name = "closed_won";
    event.properties.value = dealValue;
  } else if (stage.includes("closed lost") || stage === "lost") {
    event.properties.event_name = "closed_lost";
    event.properties.value = 0;
  }

  // Remove undefined values
  event.properties = Object.fromEntries(
    Object.entries(event.properties).filter(([_, v]) => v !== undefined)
  );

  return event;
}

function normalizePhone(phone?: string): string | null {
  if (!phone) return null;

  const digits = phone.replace(/[^\d+]/g, "");

  if (digits.startsWith("+")) {
    return digits;
  } else if (digits.length === 10) {
    return `+1${digits}`;
  } else if (digits.length === 11 && digits.startsWith("1")) {
    return `+${digits}`;
  }

  return `+${digits}`;
}

function getMonetaryValue(payload: GHLPayload): number | null {
  const value = payload.monetaryValue || payload.leadValue;

  if (value === undefined || value === null) {
    return null;
  }

  if (typeof value === "number") {
    return value;
  }

  const cleaned = String(value).replace(/[^\d.]/g, "");
  const parsed = parseFloat(cleaned);
  return isNaN(parsed) ? null : parsed;
}

export { transformGHLToTripleWhale, GHLPayload, TripleWhaleEvent };
```

---

## API Client with Retry

### Python

```python
import httpx
import asyncio
from typing import Optional


class TripleWhaleClient:
    """Async client with exponential backoff retry."""

    BASE_URL = "https://api.triplewhale.com/api/v2"
    MAX_RETRIES = 4
    RETRY_DELAYS = [2, 4, 8, 16]

    def __init__(self, api_key: str):
        self.client = httpx.AsyncClient(
            base_url=self.BASE_URL,
            headers={
                "x-api-key": api_key,
                "Content-Type": "application/json",
            },
            timeout=30.0,
        )

    async def send_event(self, event: dict) -> dict:
        """Send event with retry logic."""
        return await self._request("POST", "/data-in/event", event)

    async def send_order(self, order: dict) -> dict:
        """Send order with retry logic."""
        return await self._request("POST", "/data-in/orders", order)

    async def _request(
        self,
        method: str,
        endpoint: str,
        json: dict
    ) -> dict:
        """Make request with exponential backoff."""
        last_error = None

        for attempt in range(self.MAX_RETRIES + 1):
            try:
                response = await self.client.request(
                    method=method,
                    url=endpoint,
                    json=json,
                )

                if response.status_code == 200:
                    return response.json()

                # Don't retry auth errors
                if response.status_code in (401, 403, 422):
                    raise Exception(
                        f"API error {response.status_code}: {response.text}"
                    )

                # Retry on rate limit or server errors
                if response.status_code in (429, 500, 502, 503, 504):
                    last_error = Exception(
                        f"API error {response.status_code}"
                    )
                    if attempt < self.MAX_RETRIES:
                        await asyncio.sleep(self.RETRY_DELAYS[attempt])
                        continue

                raise Exception(
                    f"API error {response.status_code}: {response.text}"
                )

            except httpx.RequestError as e:
                last_error = e
                if attempt < self.MAX_RETRIES:
                    await asyncio.sleep(self.RETRY_DELAYS[attempt])
                    continue
                raise

        raise last_error or Exception("Max retries exceeded")

    async def close(self):
        await self.client.aclose()
```

### JavaScript/TypeScript

```typescript
class TripleWhaleClient {
  private baseUrl = "https://api.triplewhale.com/api/v2";
  private apiKey: string;
  private retryDelays = [2000, 4000, 8000, 16000];

  constructor(apiKey: string) {
    this.apiKey = apiKey;
  }

  async sendEvent(event: TripleWhaleEvent): Promise<unknown> {
    return this.request("POST", "/data-in/event", event);
  }

  async sendOrder(order: unknown): Promise<unknown> {
    return this.request("POST", "/data-in/orders", order);
  }

  private async request(
    method: string,
    endpoint: string,
    body: unknown
  ): Promise<unknown> {
    let lastError: Error | null = null;

    for (let attempt = 0; attempt <= this.retryDelays.length; attempt++) {
      try {
        const response = await fetch(`${this.baseUrl}${endpoint}`, {
          method,
          headers: {
            "x-api-key": this.apiKey,
            "Content-Type": "application/json",
          },
          body: JSON.stringify(body),
        });

        if (response.ok) {
          return response.json();
        }

        if ([401, 403, 422].includes(response.status)) {
          throw new Error(`API error ${response.status}: ${await response.text()}`);
        }

        if ([429, 500, 502, 503, 504].includes(response.status)) {
          lastError = new Error(`API error ${response.status}`);
          if (attempt < this.retryDelays.length) {
            await this.sleep(this.retryDelays[attempt]);
            continue;
          }
        }

        throw new Error(`API error ${response.status}: ${await response.text()}`);
      } catch (error) {
        lastError = error as Error;
        if (attempt < this.retryDelays.length) {
          await this.sleep(this.retryDelays[attempt]);
          continue;
        }
        throw error;
      }
    }

    throw lastError || new Error("Max retries exceeded");
  }

  private sleep(ms: number): Promise<void> {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }
}
```
