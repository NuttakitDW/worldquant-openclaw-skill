# WorldQuant Brain API Endpoints Reference

Based on audit of `automated-alphas-search/main.py` repository.

## Base URL

```
https://api.worldquantbrain.com
```

## Authentication

All endpoints require HTTP Basic Auth or session cookies from `/authentication` endpoint.

**Headers:**
```
Authorization: Basic base64(email:password)
```

Or use cached session cookies from login.

---

## API Endpoints

### 1. Session / Authentication

#### GET /authentication
**Purpose:** Validate current session and get token expiry info

**Request:**
```http
GET /authentication
```

**Response (200 OK):**
```json
{
  "token": {
    "value": "...",
    "expiry": 21600
  },
  "user": {
    "id": "user_123",
    "email": "user@example.com"
  }
}
```

**Status Codes:**
- `200` - Session valid
- `204` - Session valid (no body)
- `401` - Session expired
- `429` - Rate limited

**Used in:** Session validation, token refresh check

**Related code:**
```python
# From utils/session.py
response = session.get(API_USERS_SELF, timeout=TIMEOUT_SESSION_VALIDATE)
```

---

#### GET /users/self
**Purpose:** Get current authenticated user info

**Request:**
```http
GET /users/self
```

**Response (200 OK):**
```json
{
  "id": "user_123",
  "email": "user@example.com",
  "username": "username",
  "account_type": "expert"
}
```

**Status Codes:**
- `200` - Success
- `401` - Unauthorized
- `429` - Rate limited

---

### 2. Data Management

#### GET /data-sets
**Purpose:** List available datasets (regions, universes, delays)

**Request:**
```http
GET /data-sets?region=USA&universe=TOP3000&delay=1&limit=50&offset=0
```

**Query Parameters:**
- `region` (string) - "USA", "EUR", "ASI", "IND", "CHN", "GLB"
- `universe` (string) - "TOP3000", "TOP5000", etc.
- `delay` (integer) - 1, 2, etc.
- `limit` (integer, default: 50) - Max results per page
- `offset` (integer, default: 0) - Pagination offset

**Response (200 OK):**
```json
{
  "results": [
    {
      "id": "dataset_1",
      "name": "USA Daily Data",
      "region": "USA",
      "universe": "TOP3000",
      "delay": 1
    }
  ],
  "next": null,
  "total": 1
}
```

**Status Codes:**
- `200` - Success
- `401` - Unauthorized
- `429` - Rate limited
- `500` - Server error

**Used in:** Load available datasets, discover field options

**Related code:**
```python
def get_all_datasets(session, region='USA', universe='TOP3000', delay=1):
    url = API_DATASETS
    params = {
        'region': region,
        'universe': universe,
        'delay': delay,
        'limit': limit,
        'offset': offset
    }
    response = session.get(url, params=params, timeout=TIMEOUT_DATASETS)
```

---

#### GET /data-fields
**Purpose:** List available fields for a dataset

**Request:**
```http
GET /data-fields?dataset=dataset_1&field_type=fundamentals&search=price&limit=50
```

**Query Parameters:**
- `dataset` (string) - Dataset ID
- `field_type` (string, optional) - "fundamentals", "technicals", "derived"
- `search` (string, optional) - Search field names
- `limit` (integer, default: 50) - Max results
- `offset` (integer, default: 0) - Pagination

**Response (200 OK):**
```json
{
  "results": [
    {
      "id": "field_123",
      "name": "close_price",
      "description": "Closing price",
      "field_type": "fundamentals",
      "data_type": "double"
    }
  ],
  "total": 100
}
```

**Status Codes:**
- `200` - Success
- `401` - Unauthorized
- `429` - Rate limited
- `400` - Bad request

**Error Handling:**
```python
if response.status_code == 429:
    retry_after = int(response.headers.get('Retry-After', 60))
    # Wait and retry
```

**Used in:** Discover available fields, list operators

---

### 3. Alpha Submission & Simulation

#### POST /simulations
**Purpose:** Submit one or more alphas for backtesting

**Request:**
```http
POST /simulations
Content-Type: application/json

[
  {
    "definition": "rank(close) * volume",
    "universe": "TOP3000",
    "region": "USA",
    "delay": 1,
    "truncation": 0.08,
    "pasteurization": "ON",
    "neutralizations": [],
    "decay": 0
  }
]
```

**Request Body:** Array of alpha objects

**Alpha Object Fields:**
- `definition` (string, required) - Alpha expression
- `universe` (string) - "TOP3000", "TOP5000", etc.
- `region` (string) - "USA", "EUR", "ASI", "IND", "CHN", "GLB"
- `delay` (integer) - 1, 2, etc.
- `truncation` (float) - 0.01 - 0.1 (default: 0.08)
- `pasteurization` (string) - "ON" or "OFF" (default: "ON")
- `neutralizations` (array) - [] or ["sector"], ["market"], etc.
- `decay` (integer) - 0, 1, 2 (default: 0)

**Response (202 Accepted):**
```json
{
  "results": [
    {
      "id": "sim_abc123",
      "definition": "rank(close) * volume",
      "status": "submitted",
      "location": "/simulations/sim_abc123"
    }
  ]
}
```

**Status Codes:**
- `202` - Accepted (processing)
- `400` - Bad request (invalid alpha syntax)
- `401` - Unauthorized
- `429` - Rate limited
- `500` - Server error

**Related code:**
```python
response = session.post(
    API_SIMULATIONS,
    json=alphas,
    timeout=TIMEOUT_SIMULATION_SUBMIT
)

if response.status_code == 401:
    # Reload session and retry
    session = reload_session_from_cache(session, logger)
```

**Note:** Response is 202, not 201. Submission is asynchronous.

---

#### GET /simulations/{simulation_id}
**Purpose:** Poll for simulation results

**Request:**
```http
GET /simulations/sim_abc123
```

**Response (200 OK) - In Progress:**
```json
{
  "id": "sim_abc123",
  "status": "pending",
  "progress": 50,
  "created_at": "2026-04-26T09:30:00Z"
}
```

**Response (200 OK) - Complete:**
```json
{
  "id": "sim_abc123",
  "status": "completed",
  "definition": "rank(close) * volume",
  "metrics": {
    "sharpe": 1.45,
    "turnover": 0.52,
    "fitness": 0.88,
    "max_drawdown": -0.15,
    "return": 0.25,
    "annual_volatility": 0.18
  },
  "parent_id": "parent_123",
  "children": ["child_456", "child_789"],
  "created_at": "2026-04-26T09:30:00Z",
  "completed_at": "2026-04-26T10:00:00Z"
}
```

**Status Codes:**
- `200` - Success (either pending or completed)
- `401` - Unauthorized
- `404` - Simulation not found
- `429` - Rate limited

**Status Values:**
- `pending` - Still processing
- `completed` - Done with results
- `failed` - Error during execution
- `cancelled` - User cancelled

**Polling Strategy:**
```python
max_polls = 120  # 120 * 2s = 4 minutes max
for poll_num in range(1, max_polls + 1):
    poll_response = session.get(location_url, timeout=TIMEOUT_SIMULATION_POLL)
    
    if poll_response.status_code == 401:
        # Session expired during polling
        session = reload_session_from_cache(session, logger)
        continue
    
    if poll_response.status_code == 200:
        data = poll_response.json()
        if data.get('status') == 'completed':
            # Process results
            break
    
    time.sleep(2)  # 2-second backoff
```

---

#### GET /simulations/{simulation_id}/{child_id}
**Purpose:** Fetch child simulation results (from parent submission)

**Request:**
```http
GET /simulations/sim_parent/sim_child_123
```

**Response (200 OK):**
Same as parent simulation response with individual metrics.

**Status Codes:**
- `200` - Success
- `401` - Unauthorized (reload session)
- `404` - Child not found
- `429` - Rate limited

---

### 4. Alpha Management

#### GET /alphas
**Purpose:** List all submitted alphas (optional)

**Request:**
```http
GET /alphas?limit=50&offset=0
```

**Response (200 OK):**
```json
{
  "results": [
    {
      "id": "alpha_123",
      "definition": "rank(close) * volume",
      "status": "active",
      "metrics": {
        "sharpe": 1.45,
        "turnover": 0.52
      },
      "created_at": "2026-04-26T09:30:00Z"
    }
  ],
  "total": 50
}
```

---

#### GET /alphas/{alpha_id}
**Purpose:** Fetch specific alpha details

**Request:**
```http
GET /alphas/alpha_123
```

**Response (200 OK):**
```json
{
  "id": "alpha_123",
  "definition": "rank(close) * volume",
  "status": "active",
  "universe": "TOP3000",
  "region": "USA",
  "metrics": {
    "sharpe": 1.45,
    "turnover": 0.52,
    "fitness": 0.88
  },
  "created_at": "2026-04-26T09:30:00Z",
  "simulations": ["sim_abc123", "sim_def456"]
}
```

**Related code:**
```python
alpha_url = f'{API_ALPHAS}/{alpha_id}'
alpha_response = session.get(alpha_url, timeout=TIMEOUT_ALPHA_FETCH)
```

---

## Error Handling

### HTTP Status Codes

| Code | Meaning | Action |
|------|---------|--------|
| 200 | OK | Process response |
| 202 | Accepted | Poll for results |
| 204 | No Content | Session valid (no body) |
| 400 | Bad Request | Check alpha syntax, fix and retry |
| 401 | Unauthorized | Reload session from cache, retry |
| 404 | Not Found | Check ID, may be invalid |
| 429 | Rate Limited | Wait (check Retry-After header), then retry |
| 500 | Server Error | Retry with backoff |

### Session Reload on 401

```python
def safe_request(session, method, url, logger=None, max_retries=3, **kwargs):
    retry_count = 0
    
    while retry_count <= max_retries:
        response = session.request(method, url, **kwargs)
        
        # Auto-reload on 401
        if response.status_code == 401 and retry_count < max_retries:
            logger.warning(f"HTTP 401, reloading session...")
            session = reload_session_from_cache(session, logger)
            retry_count += 1
            continue
        
        return response
```

### Rate Limiting (429)

```python
if response.status_code == 429:
    retry_after = int(response.headers.get('Retry-After', 60))
    logger.warning(f"Rate limited! Waiting {retry_after}s...")
    time.sleep(retry_after)
    # Retry request
```

---

## Timeouts

| Operation | Timeout | Purpose |
|-----------|---------|---------|
| `TIMEOUT_SESSION_VALIDATE` | 10s | Validate session |
| `TIMEOUT_DATASETS` | 30s | Fetch datasets |
| `TIMEOUT_DATAFIELDS` | 30s | Fetch fields |
| `TIMEOUT_SIMULATION_SUBMIT` | 60s | Submit alphas to simulate |
| `TIMEOUT_SIMULATION_POLL` | 30s | Poll for results |
| `TIMEOUT_ALPHA_FETCH` | 30s | Fetch alpha details |

---

## Retry Strategy

**Config:**
```python
MAX_RETRIES = 3
INITIAL_BACKOFF = 2.0  # seconds
MAX_BACKOFF = 60.0     # seconds
BACKOFF_MULTIPLIER = 2.0
JITTER_FACTOR = 0.1    # 10% jitter
```

**Algorithm:**
```
backoff = INITIAL_BACKOFF
for attempt in range(MAX_RETRIES):
    try:
        response = session.request(method, url)
        if response.status_code < 500:
            return response
    except Exception:
        pass
    
    # Exponential backoff with jitter
    jitter = backoff * JITTER_FACTOR * random.uniform(-1, 1)
    wait_time = min(backoff + jitter, MAX_BACKOFF)
    time.sleep(wait_time)
    backoff *= BACKOFF_MULTIPLIER
```

---

## Simulation Results Schema

**Metrics returned:**

```json
{
  "id": "sim_abc123",
  "definition": "rank(close) * volume",
  "status": "completed",
  "region": "USA",
  "universe": "TOP3000",
  "delay": 1,
  "metrics": {
    "sharpe": 1.45,           // Risk-adjusted return
    "turnover": 0.52,         // Portfolio turnover (target: <70%)
    "fitness": 0.88,          // Weighted score
    "max_drawdown": -0.15,    // Worst loss
    "return": 0.25,           // Total return
    "annual_volatility": 0.18,// Risk level
    "count": 3000             // Number of assets
  },
  "parent_id": "sim_parent",
  "children": ["sim_child_1", "sim_child_2"],
  "created_at": "2026-04-26T09:30:00Z",
  "completed_at": "2026-04-26T10:00:00Z"
}
```

**Constraints (Expert tier):**
- `sharpe > 1.25`
- `turnover < 70%`
- `fitness >= 1.0`

---

## Examples

### Submit and Monitor Alpha

```python
import time
from config.settings import API_SIMULATIONS, TIMEOUT_SIMULATION_SUBMIT, TIMEOUT_SIMULATION_POLL

# 1. Submit alpha
alpha = {
    "definition": "rank(close) * volume",
    "universe": "TOP3000",
    "region": "USA",
    "delay": 1,
    "truncation": 0.08,
    "pasteurization": "ON"
}

response = session.post(
    API_SIMULATIONS,
    json=[alpha],
    timeout=TIMEOUT_SIMULATION_SUBMIT
)

if response.status_code != 202:
    print(f"Submit failed: {response.status_code}")
    exit(1)

# 2. Extract simulation ID from Location header
sim_id = response.json()['results'][0]['id']
sim_url = f"{API_SIMULATIONS}/{sim_id}"

# 3. Poll for results
max_polls = 120
for poll_num in range(1, max_polls + 1):
    poll_response = session.get(sim_url, timeout=TIMEOUT_SIMULATION_POLL)
    
    if poll_response.status_code == 200:
        data = poll_response.json()
        if data.get('status') == 'completed':
            print(f"✅ Simulation complete!")
            print(f"   Sharpe: {data['metrics']['sharpe']}")
            print(f"   Turnover: {data['metrics']['turnover']}")
            break
        else:
            print(f"⏳ Poll {poll_num}: Status {data.get('status')}...")
    
    time.sleep(2)
```

---

## Rate Limits

- **Per endpoint:** ~100 requests/minute (varies)
- **Simulation submissions:** Staggered to prevent burst
- **Response header:** `Retry-After` indicates wait time

---

## Session Management

**Expiry:** ~4-6 hours (Expert tier)
**Reload:** Automatic on 401 errors
**Cache file:** `brain_session_cache.json`

**Cache schema:**
```json
{
  "cookies": [
    {"name": "sessionid", "value": "...", "domain": "...", "path": "/"}
  ],
  "auth": ["email", "password"],
  "expiry_seconds": 21600,
  "cached_at": 1234567890.123
}
```

---

## References

- Source: `/home/node/.openclaw/workspace/automated-alphas-search/main.py`
- Config: `/config/settings.py`
- Utils: `/utils/session.py`
