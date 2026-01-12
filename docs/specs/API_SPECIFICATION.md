# API Specification
## CompeteDexter REST API v1

**Version**: 1.0.0
**Base URL**: `https://api.competedexter.com/v1`
**Authentication**: Bearer token (JWT)

---

## Authentication

### POST /auth/register

Register a new user account.

**Request:**
```json
{
  "email": "user@example.com",
  "password": "SecurePassword123!",
  "name": "John Doe"
}
```

**Response (201):**
```json
{
  "user": {
    "id": "usr_1234",
    "email": "user@example.com",
    "name": "John Doe",
    "created_at": "2026-01-12T10:00:00Z"
  },
  "token": "eyJhbGc..."
}
```

---

### POST /auth/login

Authenticate and receive JWT token.

**Request:**
```json
{
  "email": "user@example.com",
  "password": "SecurePassword123!"
}
```

**Response (200):**
```json
{
  "user": {
    "id": "usr_1234",
    "email": "user@example.com",
    "name": "John Doe"
  },
  "token": "eyJhbGc...",
  "expires_in": 604800
}
```

---

## Competitors

### POST /competitors

Add a new competitor to track.

**Request:**
```json
{
  "name": "Acme Corp",
  "domain": "acme.com",
  "description": "Leading provider of enterprise software",
  "tier": "primary",
  "tags": ["saas", "enterprise"]
}
```

**Response (201):**
```json
{
  "id": "comp_5678",
  "name": "Acme Corp",
  "domain": "acme.com",
  "description": "Leading provider of enterprise software",
  "tier": "primary",
  "tags": ["saas", "enterprise"],
  "logo_url": "https://logo.clearbit.com/acme.com",
  "created_at": "2026-01-12T10:00:00Z",
  "updated_at": "2026-01-12T10:00:00Z"
}
```

---

### GET /competitors

List all competitors in workspace.

**Query Parameters:**
- `tier` (optional): Filter by tier (primary, secondary, emerging)
- `tags` (optional): Comma-separated list of tags
- `limit` (optional): Number of results (default: 50, max: 100)
- `offset` (optional): Pagination offset

**Response (200):**
```json
{
  "data": [
    {
      "id": "comp_5678",
      "name": "Acme Corp",
      "domain": "acme.com",
      "tier": "primary",
      "tags": ["saas", "enterprise"],
      "logo_url": "https://logo.clearbit.com/acme.com"
    }
  ],
  "pagination": {
    "total": 12,
    "limit": 50,
    "offset": 0
  }
}
```

---

### GET /competitors/{id}

Get detailed information about a competitor.

**Response (200):**
```json
{
  "id": "comp_5678",
  "name": "Acme Corp",
  "domain": "acme.com",
  "description": "Leading provider of enterprise software",
  "tier": "primary",
  "tags": ["saas", "enterprise"],
  "logo_url": "https://logo.clearbit.com/acme.com",
  "metadata": {
    "founded": 2015,
    "employees": 250,
    "funding": {
      "total": 150000000,
      "last_round": "Series C",
      "last_round_date": "2025-12-15"
    }
  },
  "created_at": "2026-01-12T10:00:00Z",
  "updated_at": "2026-01-12T10:00:00Z"
}
```

---

### PUT /competitors/{id}

Update competitor information.

**Request:**
```json
{
  "description": "Updated description",
  "tier": "secondary",
  "tags": ["saas"]
}
```

**Response (200):**
```json
{
  "id": "comp_5678",
  "name": "Acme Corp",
  "description": "Updated description",
  "tier": "secondary",
  "updated_at": "2026-01-12T11:00:00Z"
}
```

---

### DELETE /competitors/{id}

Remove a competitor from tracking.

**Response (204):**
No content.

---

## Queries

### POST /query/ask

Ask a competitive intelligence question.

**Request:**
```json
{
  "query": "What is Acme Corp's pricing strategy?",
  "competitor_ids": ["comp_5678"],
  "context": {
    "conversation_id": "conv_1234"
  }
}
```

**Response (200) - Streaming:**

Server-Sent Events format:

```
data: {"type":"phase","phase":"understand"}

data: {"type":"phase","phase":"plan"}

data: {"type":"task_start","task":{"id":"task_1","name":"Fetch pricing data"}}

data: {"type":"task_complete","task":{"id":"task_1"},"result":"..."}

data: {"type":"answer","chunk":"Acme Corp uses a tiered pricing model..."}

data: {"type":"answer","chunk":" with three main tiers: Starter, Pro, and Enterprise."}

data: {"type":"done","query_id":"query_9012"}
```

---

### GET /query/history

Get query history for the user.

**Query Parameters:**
- `limit` (optional): Number of results (default: 20, max: 100)
- `offset` (optional): Pagination offset
- `competitor_id` (optional): Filter by competitor

**Response (200):**
```json
{
  "data": [
    {
      "id": "query_9012",
      "query": "What is Acme Corp's pricing strategy?",
      "answer": "Acme Corp uses a tiered pricing model...",
      "competitor_ids": ["comp_5678"],
      "sources": [
        {
          "title": "Acme Pricing Page",
          "url": "https://acme.com/pricing",
          "accessed_at": "2026-01-12T10:30:00Z"
        }
      ],
      "created_at": "2026-01-12T10:25:00Z"
    }
  ],
  "pagination": {
    "total": 45,
    "limit": 20,
    "offset": 0
  }
}
```

---

### GET /query/{id}

Get details of a specific query.

**Response (200):**
```json
{
  "id": "query_9012",
  "query": "What is Acme Corp's pricing strategy?",
  "answer": "Acme Corp uses a tiered pricing model...",
  "competitor_ids": ["comp_5678"],
  "sources": [
    {
      "title": "Acme Pricing Page",
      "url": "https://acme.com/pricing",
      "accessed_at": "2026-01-12T10:30:00Z"
    }
  ],
  "metadata": {
    "execution_time_ms": 8450,
    "tokens_used": 3200,
    "data_sources_used": ["website", "g2_reviews"]
  },
  "task_results": [
    {
      "task_id": "task_1",
      "task_name": "Fetch pricing data",
      "status": "completed",
      "result_summary": "Retrieved pricing data from website"
    }
  ],
  "created_at": "2026-01-12T10:25:00Z",
  "completed_at": "2026-01-12T10:25:08Z"
}
```

---

### GET /query/suggestions

Get suggested queries based on workspace competitors.

**Response (200):**
```json
{
  "suggestions": [
    {
      "query": "What funding did my competitors raise recently?",
      "category": "financial",
      "description": "Get latest funding rounds for all competitors"
    },
    {
      "query": "Compare pricing across competitors",
      "category": "pricing",
      "description": "Side-by-side pricing comparison"
    },
    {
      "query": "What features do competitors have that we don't?",
      "category": "product",
      "description": "Feature gap analysis"
    }
  ]
}
```

---

## Data Access

### GET /competitors/{id}/data

Get all data for a specific competitor.

**Query Parameters:**
- `type` (optional): Filter by data type (website, jobs, funding, news, etc.)
- `from_date` (optional): ISO 8601 date
- `to_date` (optional): ISO 8601 date
- `limit` (optional): Number of results (default: 50)

**Response (200):**
```json
{
  "data": [
    {
      "id": "data_1234",
      "type": "funding",
      "timestamp": "2025-12-15T00:00:00Z",
      "data": {
        "round": "Series C",
        "amount": 50000000,
        "lead_investor": "Sequoia Capital",
        "investors": ["Sequoia Capital", "a16z"]
      },
      "source_url": "https://crunchbase.com/..."
    },
    {
      "id": "data_5678",
      "type": "news",
      "timestamp": "2026-01-10T15:30:00Z",
      "data": {
        "title": "Acme Corp Launches New Product",
        "summary": "Acme Corp announced...",
        "url": "https://techcrunch.com/..."
      }
    }
  ],
  "pagination": {
    "total": 145,
    "limit": 50,
    "offset": 0
  }
}
```

---

### GET /competitors/{id}/timeline

Get activity timeline for a competitor.

**Response (200):**
```json
{
  "events": [
    {
      "id": "event_1",
      "type": "funding",
      "title": "Raised $50M Series C",
      "description": "Led by Sequoia Capital",
      "timestamp": "2025-12-15T00:00:00Z",
      "significance": "high"
    },
    {
      "id": "event_2",
      "type": "product_launch",
      "title": "Launched Mobile App",
      "description": "New iOS and Android apps",
      "timestamp": "2026-01-10T00:00:00Z",
      "significance": "medium"
    }
  ]
}
```

---

## Alerts

### GET /alerts

List alerts for the workspace.

**Query Parameters:**
- `read` (optional): Filter by read status (true/false)
- `severity` (optional): Filter by severity (low, medium, high, critical)
- `limit` (optional): Number of results (default: 50)

**Response (200):**
```json
{
  "data": [
    {
      "id": "alert_1234",
      "type": "funding",
      "competitor_id": "comp_5678",
      "competitor_name": "Acme Corp",
      "title": "Acme Corp raised $50M Series C",
      "description": "Led by Sequoia Capital...",
      "severity": "high",
      "read": false,
      "created_at": "2025-12-15T10:00:00Z"
    }
  ],
  "pagination": {
    "total": 23,
    "unread": 8,
    "limit": 50,
    "offset": 0
  }
}
```

---

### PUT /alerts/{id}/read

Mark an alert as read.

**Response (200):**
```json
{
  "id": "alert_1234",
  "read": true,
  "read_at": "2026-01-12T10:00:00Z"
}
```

---

### POST /alerts/rules

Create a new alert rule.

**Request:**
```json
{
  "competitor_id": "comp_5678",
  "rule_type": "funding",
  "conditions": {
    "min_amount": 10000000
  },
  "channels": ["email", "slack", "in_app"]
}
```

**Response (201):**
```json
{
  "id": "rule_1234",
  "competitor_id": "comp_5678",
  "rule_type": "funding",
  "conditions": {
    "min_amount": 10000000
  },
  "channels": ["email", "slack", "in_app"],
  "enabled": true,
  "created_at": "2026-01-12T10:00:00Z"
}
```

---

### GET /alerts/rules

List all alert rules.

**Response (200):**
```json
{
  "data": [
    {
      "id": "rule_1234",
      "competitor_id": "comp_5678",
      "competitor_name": "Acme Corp",
      "rule_type": "funding",
      "conditions": {
        "min_amount": 10000000
      },
      "channels": ["email", "slack"],
      "enabled": true
    }
  ]
}
```

---

## Rate Limiting

Rate limits are based on subscription plan:

- **Free**: 20 queries/month
- **Starter**: 1,000 queries/month
- **Professional**: 10,000 queries/month
- **Enterprise**: Unlimited

**Rate Limit Headers:**
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 973
X-RateLimit-Reset: 1704096000
```

**429 Too Many Requests:**
```json
{
  "error": "Rate limit exceeded",
  "message": "You have used 1000/1000 queries this month",
  "reset_at": "2026-02-01T00:00:00Z"
}
```

---

## Error Responses

### 400 Bad Request
```json
{
  "error": "Invalid request",
  "message": "Missing required field: name",
  "details": {
    "field": "name",
    "expected": "string"
  }
}
```

### 401 Unauthorized
```json
{
  "error": "Unauthorized",
  "message": "Invalid or expired token"
}
```

### 404 Not Found
```json
{
  "error": "Not found",
  "message": "Competitor with id comp_5678 not found"
}
```

### 500 Internal Server Error
```json
{
  "error": "Internal server error",
  "message": "An unexpected error occurred",
  "request_id": "req_abc123"
}
```

---

## Webhooks (Enterprise)

Subscribe to events via webhooks.

**Supported Events:**
- `competitor.created`
- `competitor.updated`
- `alert.created`
- `funding.detected`
- `product_launch.detected`
- `pricing_change.detected`

**Webhook Payload:**
```json
{
  "event": "funding.detected",
  "timestamp": "2026-01-12T10:00:00Z",
  "data": {
    "competitor_id": "comp_5678",
    "competitor_name": "Acme Corp",
    "round": "Series C",
    "amount": 50000000
  }
}
```

---

**API Version**: 1.0.0
**Last Updated**: 2026-01-12
**OpenAPI Spec**: [Download openapi.yaml](./openapi.yaml)
