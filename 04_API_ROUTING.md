# API Routing Design
## ElectricSQL + FastAPI Integration

---

## 1. API Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    API LAYER ARCHITECTURE                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  CLIENT                                                      │
│    │                                                         │
│    ├─────► /api/* ─────► FastAPI Backend ─────► PostgreSQL   │
│    │       (Business Logic, Validation)                      │
│    │                                                         │
│    └─────► /v1/shape/* ─────► Electric ─────► PostgreSQL     │
│            (Sync Operations via WebSocket/HTTP)              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Endpoint Categories

### 2.1 Authentication Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/auth/login` | Login, returns JWT |
| POST | `/api/auth/refresh` | Refresh JWT token |
| POST | `/api/auth/logout` | Invalidate token |
| GET | `/api/auth/me` | Get current user info |

### 2.2 Electric Sync Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/shape` | Get shape definition |
| GET | `/v1/shape/{shape_id}` | Subscribe to shape (SSE/WS) |
| POST | `/v1/shape/{shape_id}/sync` | Initial sync for shape |
| WS | `/v1/shape/{shape_id}/live` | Real-time sync WebSocket |

### 2.3 Business Logic Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/barcode/scan` | Validate & process barcode scan |
| POST | `/api/barcode/pair` | Pair barcode to parent |
| GET | `/api/batch/{id}` | Get batch details |
| POST | `/api/batch/{id}/activate` | Activate batch |

### 2.4 Sync Status Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/sync/status` | Get sync status |
| POST | `/api/sync/force` | Force full resync |
| GET | `/api/sync/pending` | Get pending operations count |

---

## 3. Shape Definitions

### 3.1 Shape: vendor_master

```yaml
# Subscription URL: /v1/shape/vendor_master?vendor_code=ABC
shape:
  name: vendor_master
  tables:
    - bc_batch
    - bc_config
    - bc_mfg
  params:
    - vendor_code
  where: |
    vendor_code = :vendor_code 
    AND is_deleted = false
  permissions:
    read: vendor_member
    write: vendor_admin
```

### 3.2 Shape: batch_data

```yaml
# Subscription URL: /v1/shape/batch_data?batch_id=123
shape:
  name: batch_data
  tables:
    - bc_parent
    - bc_barcode
    - bc_token
  params:
    - batch_id
  where: |
    batch_id = :batch_id 
    AND is_deleted = false
  permissions:
    read: batch_assigned
    write: batch_operator
```

### 3.3 Shape: pairing_logs

```yaml
# Subscription URL: /v1/shape/pairing_logs?vendor_code=ABC
shape:
  name: pairing_logs
  tables:
    - bc_pair_brcdxparent
    - bc_inbound_pairing
  params:
    - vendor_code
  where: |
    vendor_code = :vendor_code 
    AND created_at > now() - interval '7 days'
  permissions:
    read: vendor_member
    write: batch_operator
```

---

## 4. Request/Response Flow

### 4.1 Initial Sync Flow

```
Client                    API Gateway              Electric              PostgreSQL
  │                           │                       │                      │
  │ 1. POST /api/auth/login   │                       │                      │
  │──────────────────────────►│                       │                      │
  │                           │ validate credentials  │                      │
  │                           │──────────────────────────────────────────────►│
  │                           │                       │                      │
  │ ◄──────────────────────── │ JWT token             │                      │
  │                           │                       │                      │
  │ 2. GET /v1/shape/vendor_master?vendor_code=ABC    │                      │
  │──────────────────────────►│──────────────────────►│                      │
  │                           │                       │ query shape data     │
  │                           │                       │─────────────────────►│
  │                           │                       │                      │
  │ ◄─────────────────────────────────────────────────│ stream data chunks   │
  │ (chunked response)        │                       │                      │
  │                           │                       │                      │
  │ 3. WS /v1/shape/vendor_master/live                │                      │
  │──────────────────────────►│──────────────────────►│                      │
  │ ◄═════════════════════════════════════════════════│ WebSocket connected  │
  │ (real-time updates)       │                       │                      │
```

### 4.2 Barcode Scan Flow (Online)

```
Client                    FastAPI                  Electric              PostgreSQL
  │                           │                       │                      │
  │ 1. Scan barcode locally   │                       │                      │
  │ (write to SQLite)         │                       │                      │
  │                           │                       │                      │
  │ 2. POST /api/barcode/scan │                       │                      │
  │ {barcode: "ABC123"}       │                       │                      │
  │──────────────────────────►│                       │                      │
  │                           │ validate barcode      │                      │
  │                           │──────────────────────────────────────────────►│
  │                           │                       │                      │
  │                           │ log to bc_logs        │                      │
  │                           │──────────────────────────────────────────────►│
  │                           │                       │                      │
  │                           │                       │ WAL trigger          │
  │                           │                       │◄─────────────────────│
  │                           │                       │                      │
  │ ◄──────────────────────── │ validation result     │                      │
  │                           │                       │                      │
  │ ◄═════════════════════════════════════════════════│ sync update to other │
  │ (WebSocket update)        │                       │ devices              │
```

### 4.3 Offline → Online Sync Flow

```
Client (comes online)     FastAPI                  Electric              PostgreSQL
  │                           │                       │                      │
  │ 1. GET /v1/shape/batch_data?since=<checkpoint>    │                      │
  │──────────────────────────►│──────────────────────►│                      │
  │                           │                       │ fetch deltas         │
  │                           │                       │─────────────────────►│
  │                           │                       │                      │
  │ ◄─────────────────────────────────────────────────│ return changes       │
  │                           │                       │                      │
  │ (CRDT merge locally)      │                       │                      │
  │                           │                       │                      │
  │ 2. POST /v1/shape/batch_data/write                │                      │
  │ {operations: [...]}       │                       │                      │
  │──────────────────────────►│──────────────────────►│                      │
  │                           │                       │ apply CRDT ops       │
  │                           │                       │─────────────────────►│
  │                           │                       │                      │
  │ ◄─────────────────────────────────────────────────│ confirm              │
```

---

## 5. Authentication & Authorization

### 5.1 JWT Token Structure

```json
{
  "sub": "user_123",
  "vendor_code": "ABC",
  "roles": ["operator", "viewer"],
  "assigned_batches": [1, 2, 3],
  "exp": 1735200000,
  "iat": 1735113600
}
```

### 5.2 Permission Model

| Role | Shape Access | Write Access |
|------|--------------|--------------|
| `viewer` | vendor_master (read) | None |
| `operator` | batch_data (read/write) | bc_barcode, bc_pair_* |
| `admin` | All shapes | All tables |

### 5.3 Row-Level Security

```sql
-- Electric applies these filters automatically
-- Users can only see data for their vendor_code

POLICY vendor_isolation ON bc_barcode
  USING (vendor_code = current_setting('app.vendor_code'));

POLICY batch_assignment ON bc_barcode
  USING (batch_id IN (
    SELECT batch_id FROM user_batch_assignments 
    WHERE user_id = current_setting('app.user_id')
  ));
```

---

## 6. Error Handling

### 6.1 Error Response Format

```json
{
  "error": {
    "code": "SYNC_CONFLICT",
    "message": "Barcode already paired to different parent",
    "details": {
      "barcode_id": 12345,
      "current_parent": 500,
      "attempted_parent": 600
    },
    "retry_after": null
  }
}
```

### 6.2 Error Codes

| Code | HTTP | Description |
|------|------|-------------|
| `AUTH_EXPIRED` | 401 | Token expired, refresh needed |
| `AUTH_INVALID` | 401 | Invalid token |
| `PERMISSION_DENIED` | 403 | No access to resource |
| `SYNC_CONFLICT` | 409 | CRDT conflict (manual resolution needed) |
| `SHAPE_NOT_FOUND` | 404 | Shape definition not found |
| `RATE_LIMITED` | 429 | Too many requests |
| `SYNC_OUTDATED` | 410 | Shape version outdated, resync needed |

---

## 7. Rate Limiting

| Endpoint Type | Limit | Window |
|---------------|-------|--------|
| Auth endpoints | 10 req | 1 min |
| Shape sync | 100 req | 1 min |
| Business logic | 1000 req | 1 min |
| WebSocket | 1 connection | per device |

---

## 8. WebSocket Protocol

### 8.1 Connection

```javascript
// Client connects with JWT in query param
ws = new WebSocket('/v1/shape/batch_data/live?token=<jwt>&batch_id=123')
```

### 8.2 Message Types

**Server → Client:**
```json
// Data update
{"type": "data", "table": "bc_barcode", "op": "UPDATE", "row": {...}}

// Checkpoint
{"type": "checkpoint", "lsn": "0/ABC123"}

// Heartbeat
{"type": "heartbeat", "ts": 1735200000}
```

**Client → Server:**
```json
// ACK checkpoint
{"type": "ack", "lsn": "0/ABC123"}

// Push operation
{"type": "write", "table": "bc_barcode", "op": "UPDATE", "row": {...}}
```

