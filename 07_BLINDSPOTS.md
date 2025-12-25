# Blindspots & Risk Analysis
## Edge Cases, Risks, and Mitigation Strategies

---

## 1. Risk Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    RISK MATRIX                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  IMPACT                                                      │
│    ▲                                                         │
│    │                                                         │
│  H │  ┌────────┐     ┌────────┐                             │
│  I │  │ Clock  │     │ Data   │                             │
│  G │  │ Drift  │     │ Loss   │                             │
│  H │  └────────┘     └────────┘                             │
│    │                                                         │
│  M │  ┌────────┐     ┌────────┐     ┌────────┐             │
│  E │  │ Schema │     │Storage │     │Conflict│             │
│  D │  │ Change │     │ Full   │     │ Flood  │             │
│    │  └────────┘     └────────┘     └────────┘             │
│    │                                                         │
│  L │  ┌────────┐     ┌────────┐                             │
│  O │  │ Slow   │     │ User   │                             │
│  W │  │ Sync   │     │ Error  │                             │
│    │  └────────┘     └────────┘                             │
│    │                                                         │
│    └────────────────────────────────────────────►           │
│       LOW            MEDIUM           HIGH                   │
│                    LIKELIHOOD                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Technical Blindspots

### 2.1 Clock Drift Issue

```
┌─────────────────────────────────────────────────────────────┐
│  BLINDSPOT: Device Clock Drift                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM:                                                    │
│  • CRDT uses timestamps for conflict resolution             │
│  • If device clock is wrong, wrong data wins                │
│  • Mobile devices can have significantly wrong clocks        │
│                                                              │
│  SCENARIO:                                                   │
│  Device A clock: 2025-12-25 10:00:00 (correct)              │
│  Device B clock: 2025-12-26 10:00:00 (1 day ahead!)         │
│                                                              │
│  Both update same barcode → Device B always wins            │
│  (even for older changes)                                    │
│                                                              │
│  IMPACT: HIGH                                                │
│  LIKELIHOOD: MEDIUM                                          │
│                                                              │
│  MITIGATION:                                                 │
│  1. Use Hybrid Logical Clock (HLC) instead of wall clock    │
│  2. Validate device time delta on sync                       │
│  3. Warn user if clock is significantly off                  │
│  4. Server can reject ops with unrealistic timestamps        │
│                                                              │
│  IMPLEMENTATION:                                             │
│  • On sync connect, compare device time vs server time       │
│  • If |delta| > 5 minutes, warn user                        │
│  • If |delta| > 1 hour, block sync until fixed              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Storage Exhaustion

```
┌─────────────────────────────────────────────────────────────┐
│  BLINDSPOT: Client Storage Full                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM:                                                    │
│  • Mobile devices have limited storage                       │
│  • SQLite DB can grow large with many barcodes               │
│  • System may prevent writes when storage is low            │
│                                                              │
│  SCENARIO:                                                   │
│  • User syncs large batch (100K barcodes)                   │
│  • Phone storage at 95%                                     │
│  • SQLite write fails → sync stuck                          │
│  • User cannot scan new barcodes                            │
│                                                              │
│  IMPACT: HIGH                                                │
│  LIKELIHOOD: MEDIUM                                          │
│                                                              │
│  MITIGATION:                                                 │
│  1. Monitor storage before sync                              │
│  2. Implement LRU eviction for old batch data               │
│  3. Compress local DB periodically (VACUUM)                 │
│  4. Warn user when storage < 500MB free                     │
│  5. Allow partial sync (critical data only)                 │
│                                                              │
│  STORAGE ESTIMATION:                                         │
│  • 100K barcodes × 500 bytes = 50 MB                        │
│  • + indexes + overhead = ~100 MB per batch                 │
│  • Reserved minimum: 200 MB for operations                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 Schema Migration

```
┌─────────────────────────────────────────────────────────────┐
│  BLINDSPOT: Schema Changes During Active Sync                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM:                                                    │
│  • Server schema changes (new column, new table)            │
│  • Client has old schema in SQLite                          │
│  • Sync breaks or data is lost                              │
│                                                              │
│  SCENARIO:                                                   │
│  1. Server adds new column: bc_barcode.quality_score        │
│  2. Old client tries to sync                                │
│  3. Client receives row with unknown column                 │
│  4. Insert fails or column is silently dropped              │
│                                                              │
│  IMPACT: HIGH                                                │
│  LIKELIHOOD: LOW (planning reduces this)                    │
│                                                              │
│  MITIGATION:                                                 │
│  1. Version the sync protocol                               │
│  2. Client reports its schema version on connect             │
│  3. Server downgrades response for old clients              │
│  4. Force app update for breaking schema changes            │
│  5. Use backward-compatible migrations only                 │
│                                                              │
│  MIGRATION RULES:                                            │
│  ✅ ADD column with default value                           │
│  ✅ ADD new table                                           │
│  ⚠️ RENAME column (requires version gate)                  │
│  ❌ DROP column (never, use is_deprecated flag)             │
│  ❌ CHANGE column type (never)                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.4 Conflict Flood

```
┌─────────────────────────────────────────────────────────────┐
│  BLINDSPOT: Mass Conflict During Reconnect                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM:                                                    │
│  • Many users offline simultaneously (network outage)       │
│  • All make changes to same data                            │
│  • All reconnect at same time                               │
│  • Massive conflict resolution load                         │
│                                                              │
│  SCENARIO:                                                   │
│  • Factory WiFi down for 4 hours                            │
│  • 50 operators scanning barcodes offline                   │
│  • WiFi restored → all sync at once                         │
│  • 50,000 CRDT operations to merge                          │
│  • Server overwhelmed, sync fails                           │
│                                                              │
│  IMPACT: MEDIUM                                              │
│  LIKELIHOOD: MEDIUM                                          │
│                                                              │
│  MITIGATION:                                                 │
│  1. Jitter reconnection (random delay 0-30 sec)             │
│  2. Rate limit sync operations per client                   │
│  3. Priority queue (older offline clients first)            │
│  4. Circuit breaker on Electric service                     │
│  5. Auto-scale pods on connection spike                     │
│                                                              │
│  IMPLEMENTATION:                                             │
│  reconnect_delay = random(0, 30) + (offline_duration / 100) │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Business Logic Blindspots

### 3.1 Double Pairing Prevention

```
┌─────────────────────────────────────────────────────────────┐
│  BLINDSPOT: Barcode Paired to Multiple Parents               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM:                                                    │
│  • Business rule: 1 barcode = 1 parent only                 │
│  • Two users offline pair same barcode to different parents │
│  • CRDT merges both → barcode has 2 parents                 │
│  • Inventory count incorrect                                │
│                                                              │
│  IMPACT: HIGH (business data integrity)                     │
│  LIKELIHOOD: MEDIUM                                          │
│                                                              │
│  MITIGATION:                                                 │
│  1. First-write-wins for pairing (not LWW)                  │
│  2. Server validates pairing on sync                        │
│  3. Reject second pairing, notify user                      │
│  4. Add "pairing conflict" queue for manual resolution      │
│                                                              │
│  DETECTION:                                                  │
│  • CHECK constraint: bc_barcode.parent_id is unique per row │
│  • Periodic audit: SELECT barcodes with >1 pairing log      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Deleted Data Resurrection

```
┌─────────────────────────────────────────────────────────────┐
│  BLINDSPOT: Deleted Items Coming Back                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM:                                                    │
│  • Admin deletes barcode on server                          │
│  • Offline user edits same barcode                          │
│  • User syncs → barcode "resurrects"                        │
│                                                              │
│  SCENARIO:                                                   │
│  T1: Admin sets is_deleted = true (server)                  │
│  T2: Offline user updates note (local, timestamp after T1)  │
│  T3: User syncs → LWW picks user's update (T2 > T1)         │
│  T4: is_deleted reverted to false! 💀                       │
│                                                              │
│  IMPACT: MEDIUM                                              │
│  LIKELIHOOD: LOW                                             │
│                                                              │
│  MITIGATION:                                                 │
│  1. Use tombstone semantics (delete = permanent)            │
│  2. is_deleted uses OR-set (true always wins)               │
│  3. Separate delete operation from update                   │
│  4. Server rejects updates to deleted rows                  │
│                                                              │
│  CRDT RULE:                                                  │
│  is_deleted field uses "add-wins" semantics:                │
│  DELETE(T1) + UPDATE(T2) = DELETED (regardless of T1 vs T2) │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 Stale Read Issues

```
┌─────────────────────────────────────────────────────────────┐
│  BLINDSPOT: Acting on Stale Data                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM:                                                    │
│  • User views data that's outdated                          │
│  • Makes decision based on stale info                       │
│  • Sync updates show different reality                      │
│                                                              │
│  SCENARIO:                                                   │
│  • User A sees: "Parent X has 5 barcodes paired"            │
│  • Actually server shows: "Parent X has 10 barcodes"        │
│  • User A pairs 5 more (thinks they're completing it)       │
│  • Sync shows Parent X now has 15 (overfilled!)             │
│                                                              │
│  IMPACT: MEDIUM                                              │
│  LIKELIHOOD: MEDIUM                                          │
│                                                              │
│  MITIGATION:                                                 │
│  1. Show "last synced" timestamp prominently                │
│  2. Warn user if data is > 5 min stale                      │
│  3. For critical operations, require fresh sync             │
│  4. Use optimistic locking with version check               │
│                                                              │
│  UX RECOMMENDATION:                                          │
│  "Data from 2 hours ago" warning banner                      │
│  "Sync now" button for critical screens                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Operational Blindspots

### 4.1 Long Offline Period

```
┌─────────────────────────────────────────────────────────────┐
│  BLINDSPOT: Client Offline for Weeks                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM:                                                    │
│  • Client offline for extended period (weeks)               │
│  • Massive accumulated delta on server                      │
│  • Schema may have changed                                   │
│  • Initial re-sync takes very long or fails                 │
│                                                              │
│  IMPACT: MEDIUM                                              │
│  LIKELIHOOD: LOW                                             │
│                                                              │
│  MITIGATION:                                                 │
│  1. Track last sync timestamp per device                    │
│  2. If > 7 days offline, force full re-sync                 │
│  3. Warn user before sync: "Large sync required"            │
│  4. Allow background sync with progress                     │
│  5. Preserve local changes during re-sync                   │
│                                                              │
│  THRESHOLD LOGIC:                                            │
│  if (offline_duration < 7 days):                            │
│      incremental_sync()                                      │
│  else:                                                       │
│      warn_user("Large sync required")                        │
│      full_resync_with_merge()                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Electric Service Memory Leak

```
┌─────────────────────────────────────────────────────────────┐
│  BLINDSPOT: Memory Leak in Long-Running Service              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM:                                                    │
│  • Electric service runs 24/7                               │
│  • Memory slowly increases over days/weeks                  │
│  • Eventually OOM kills the service                         │
│  • Clients lose sync connection                             │
│                                                              │
│  IMPACT: HIGH (service outage)                              │
│  LIKELIHOOD: LOW                                             │
│                                                              │
│  MITIGATION:                                                 │
│  1. Monitor memory usage over time                          │
│  2. Set K8s memory limits (hard cap)                        │
│  3. Automatic pod rolling restart weekly                    │
│  4. Alert on memory growth trend                            │
│  5. Profile service under load                              │
│                                                              │
│  K8s CONFIG:                                                 │
│  resources:                                                  │
│    limits:                                                   │
│      memory: 4Gi                                             │
│    requests:                                                 │
│      memory: 2Gi                                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. User Experience Blindspots

### 5.1 Sync Progress UX

```
┌─────────────────────────────────────────────────────────────┐
│  BLINDSPOT: User Doesn't Know Sync State                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM:                                                    │
│  • User doesn't know if they're synced or not               │
│  • Makes changes thinking they're saved to server           │
│  • Logs out or uninstalls app                               │
│  • Changes lost (were still in local queue)                 │
│                                                              │
│  IMPACT: HIGH (user trust, data loss)                       │
│  LIKELIHOOD: MEDIUM                                          │
│                                                              │
│  MITIGATION:                                                 │
│  1. Always show sync status indicator                       │
│  2. Show pending changes count                              │
│  3. Warn before logout if pending changes exist             │
│  4. Block uninstall if pending changes (if possible)        │
│  5. Regular "all synced" notification                       │
│                                                              │
│  UX ELEMENTS:                                                │
│  ┌──────────────────────────┐                               │
│  │ ✓ All synced             │  (green)                      │
│  │ ↻ Syncing 5 items...     │  (blue, animated)             │
│  │ ⚠ 3 pending changes      │  (yellow)                     │
│  │ ✗ Offline - 12 pending   │  (red)                        │
│  └──────────────────────────┘                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Conflict Notification

```
┌─────────────────────────────────────────────────────────────┐
│  BLINDSPOT: User Unaware of Auto-Resolved Conflicts          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROBLEM:                                                    │
│  • CRDT auto-resolves conflicts                             │
│  • User's changes might be "lost" (LWW picked other)        │
│  • User doesn't know their change didn't win                │
│  • Confusion when data differs from what they entered       │
│                                                              │
│  IMPACT: MEDIUM (user confusion)                            │
│  LIKELIHOOD: MEDIUM                                          │
│                                                              │
│  MITIGATION:                                                 │
│  1. Log all conflict resolutions                            │
│  2. Notify user when their change was superseded            │
│  3. Show conflict history for debugging                     │
│  4. For critical fields, require manual resolution          │
│                                                              │
│  NOTIFICATION:                                               │
│  "Your note was updated by another user (John, 5 min ago).  │
│   Your version: 'QC OK'                                      │
│   Current version: 'QC Failed - see supervisor'"             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Risk Mitigation Summary

| Blindspot | Severity | Mitigation Status |
|-----------|----------|-------------------|
| Clock Drift | HIGH | ⚠️ Need HLC implementation |
| Storage Full | HIGH | ⚠️ Need LRU eviction |
| Schema Migration | HIGH | ✅ Version protocol |
| Conflict Flood | MEDIUM | ⚠️ Need jitter + rate limit |
| Double Pairing | HIGH | ⚠️ Need server validation |
| Deleted Resurrection | MEDIUM | ✅ OR-set semantics |
| Stale Read | MEDIUM | ⚠️ Need staleness indicator |
| Long Offline | MEDIUM | ✅ Force re-sync logic |
| Memory Leak | HIGH | ✅ K8s limits + restart |
| Sync State UX | HIGH | ⚠️ Need status indicator |
| Conflict UX | MEDIUM | ⚠️ Need notifications |

---

## 7. Recommended Pre-Launch Checklist

### Critical (Block Launch)

- [ ] HLC implementation for timestamps
- [ ] Server-side pairing validation
- [ ] Sync status indicator in UI
- [ ] Storage monitoring + warnings
- [ ] Reconnection jitter algorithm

### High Priority (Launch Within Week)

- [ ] LRU eviction for old batch data
- [ ] Conflict notification system
- [ ] Staleness warning UI
- [ ] Memory monitoring dashboard

### Nice to Have

- [ ] Conflict history viewer
- [ ] Advanced analytics on sync patterns
- [ ] Custom conflict resolution UI

