# Sync Algorithm
## ElectricSQL CRDT-Based Synchronization

---

## 1. Core Principles

```
┌─────────────────────────────────────────────────────────────┐
│                    SYNC ALGORITHM PRINCIPLES                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. LOCAL-FIRST: All reads/writes go to local DB first      │
│  2. EVENTUAL CONSISTENCY: All replicas converge             │
│  3. CONFLICT-FREE: CRDT auto-merge, no manual resolution    │
│  4. PARTIAL REPLICATION: Only sync what user needs          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. CRDT Types Used

| Field Type | CRDT | Resolution |
|------------|------|------------|
| `note`, `status` | LWW-Register | Latest timestamp wins |
| `is_deleted`, `is_archived` | Flag (OR) | True always wins |
| Pairing logs | OR-Set | All pairings kept, deduped |
| Counters | G-Counter | Sum all increments |

### CRDT Merge Example

```
Device A (Offline): barcode.note = "QC OK" @ T1
Device B (Offline): barcode.parent_id = 500 @ T2

[Both come online → CRDT Merge]

Result: { note: "QC OK", parent_id: 500 }
✅ Both changes preserved, no conflict!
```

---

## 3. Sync Phases

```
PHASE 1: BOOTSTRAP (First-time)
├── Authenticate → Get JWT
├── Subscribe vendor_master shape
├── Download bc_batch, bc_config
├── For each assigned batch:
│   └── Subscribe batch_data shape
└── Mark bootstrap complete

PHASE 2: REAL-TIME (Continuous)
├── WebSocket receives CRDT op from Electric
├── Apply to local SQLite
├── Update UI reactively
└── Send ACK

PHASE 3: PUSH (Local → Server)
├── Write to local SQLite (immediate)
├── Queue CRDT operation
├── If online: Push immediately
└── If offline: Queue for later

PHASE 4: CATCHUP (After offline)
├── Get server checkpoint
├── Fetch ops since local checkpoint
├── CRDT merge with local
└── Push pending ops
```

---

## 4. Batching Strategy

| Tier | Data | Rows | Sync Timing |
|------|------|------|-------------|
| CRITICAL | bc_config, bc_batch (active) | ~1K | Always, on app start |
| WORKING SET | bc_parent, bc_barcode per batch | ~21K/batch | On-demand |
| HISTORICAL | Completed batches | Variable | Background, low priority |
| NEVER SYNC | bc_logs, views | N/A | Server-only |

### Chunked Download

```
CHUNK_SIZE = 10000 rows
MAX_CONCURRENT = 3 chunks

FOR each chunk:
  1. Fetch chunk from Electric
  2. Write to SQLite in transaction
  3. Emit progress update
```

---

## 5. Offline Queue

```sql
CREATE TABLE pending_operations (
  id INTEGER PRIMARY KEY,
  operation_type TEXT,
  table_name TEXT,
  row_id INTEGER,
  crdt_operation BLOB,
  created_at TIMESTAMP,
  retry_count INTEGER DEFAULT 0,
  status TEXT DEFAULT 'pending'
);
```

### Queue Processing

```
FUNCTION process_pending_queue():
  pending = SELECT * FROM pending_operations WHERE status='pending'
  
  FOR op IN pending:
    TRY:
      electric.push(op.crdt_operation)
      DELETE op
    CATCH NetworkError:
      BREAK  // Stop, still offline
    CATCH Error:
      IF retry_count >= 5:
        status = 'failed'
      ELSE:
        retry_count += 1
```

---

## 6. Conflict Resolution Rules

| Table | Field | Rule |
|-------|-------|------|
| bc_barcode | parent_id | First-write-wins (one barcode = one parent) |
| bc_barcode | note | Last-write-wins |
| bc_barcode | is_deleted | True always wins |
| bc_pair_* | (row) | OR-Set, keep all unique pairings |

### Business Rule: Barcode Pairing Conflict

```
IF concurrent pairing to different parents:
  1. Keep FIRST pairing (earlier timestamp)
  2. Reject second pairing
  3. Notify user of rejection
```

---

## 7. Integrity Check

```
FUNCTION integrity_check(vendor_code):
  local = COUNT(*), SUM(id), MAX(updated_at) FROM bc_barcode
  server = electric.get_checksum(bc_barcode, vendor_code)
  
  IF local != server:
    force_resync(vendor_code)
    
Schedule: Every 6 hours when on WiFi
```

---

## 8. Complexity Analysis

| Operation | Time | Space |
|-----------|------|-------|
| Initial sync | O(N/C) | O(C) per chunk |
| Incremental sync | O(Δ) | O(Δ) |
| Single write | O(1) | O(1) |
| CRDT merge | O(1) | O(1) |

**Key insight**: Incremental sync is proportional to CHANGES, not total data.

