# P2P / LAN Sync
## Device-to-Device Synchronization on Local Network

---

## 1. Konsep P2P Sync

```
┌─────────────────────────────────────────────────────────────┐
│                    P2P SYNC CONCEPT                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  SCENARIO:                                                   │
│  • 2 operator di warehouse yang sama                        │
│  • Terhubung ke WiFi yang sama                              │
│  • Server/internet tidak stabil atau tidak ada              │
│  • Butuh sync antar device untuk pairing                    │
│                                                              │
│  SOLUTION: P2P SYNC via LAN                                  │
│                                                              │
│  ┌──────────┐      WiFi LAN       ┌──────────┐              │
│  │ Device A │◄──────────────────►│ Device B │              │
│  │ (Scan    │    Direct sync     │ (Scan    │              │
│  │  barcode)│    no server!      │  parent) │              │
│  └──────────┘                     └──────────┘              │
│                                                              │
│  HYBRID MODEL:                                               │
│  1. Primary: Sync via Electric (internet)                   │
│  2. Fallback: P2P sync via LAN (when offline)               │
│  3. Eventually: All sync to server when internet back       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Use Cases

### 2.1 Warehouse Pairing Scenario

```
┌─────────────────────────────────────────────────────────────┐
│              PAIRING USE CASE                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  SETUP:                                                      │
│  • Device A: Scan individual barcodes                       │
│  • Device B: Scan parent boxes                              │
│  • Both on same WiFi, internet slow/unavailable             │
│                                                              │
│  WORKFLOW:                                                   │
│  1. Device B scans Parent-001 (creates parent record)       │
│  2. P2P sync: Device A receives Parent-001                  │
│  3. Device A scans BC-001, pairs to Parent-001              │
│  4. P2P sync: Device B sees BC-001 paired                   │
│  5. Both devices have consistent view                       │
│  6. When internet back → both sync to server                │
│                                                              │
│  TIMELINE:                                                   │
│                                                              │
│  Device B    Device A    Server                              │
│     │           │           │                                │
│     │──(scan)──►│           │   P2P                         │
│     │◄──────────│           │   sync                        │
│     │           │──(scan)──►│                               │
│     │◄──────────│           │                               │
│     │           │           │                                │
│     │═══════════════════════│   Internet                    │
│     │           │           │   restored                    │
│     │──────────────────────►│                               │
│     │           │──────────►│                               │
│     │           │           │                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Field Operations

```
SCENARIO: Event/Exhibition venue
- No reliable internet
- 5 operators scanning products
- Need real-time inventory updates

P2P SOLUTION:
- One device acts as "local hub"
- Other devices sync to local hub
- Hub syncs to server when internet available
```

---

## 3. Technical Architecture

### 3.1 Discovery Protocol

```
┌─────────────────────────────────────────────────────────────┐
│              DEVICE DISCOVERY                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  OPTION A: mDNS/Bonjour (Recommended)                       │
│  ────────────────────────────────────                        │
│  • Standard protocol for LAN discovery                      │
│  • Works on iOS, Android, Desktop                           │
│  • No configuration needed                                  │
│                                                              │
│  Device A broadcasts:                                        │
│    _barcode-sync._tcp.local                                  │
│    Port: 8765                                                │
│    TXT: vendor=ABC, batch=123                               │
│                                                              │
│  Device B discovers:                                         │
│    "Found device at 192.168.1.100:8765"                     │
│    Same vendor? Same batch? → Connect                        │
│                                                              │
│  ─────────────────────────────────────────────               │
│                                                              │
│  OPTION B: Broadcast UDP                                     │
│  ────────────────────────                                    │
│  • Send discovery packet to 255.255.255.255                 │
│  • Simpler implementation                                   │
│  • May not work on some networks                            │
│                                                              │
│  ─────────────────────────────────────────────               │
│                                                              │
│  OPTION C: QR Code Pairing                                   │
│  ─────────────────────────                                   │
│  • Device A shows QR with IP:PORT                           │
│  • Device B scans QR to connect                             │
│  • Most reliable, no auto-discovery                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Sync Protocol

```
┌─────────────────────────────────────────────────────────────┐
│              P2P SYNC PROTOCOL                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  CONNECTION:                                                 │
│  • WebSocket over LAN                                       │
│  • TLS with self-signed cert (peer verification)            │
│  • JWT auth (same token as server)                          │
│                                                              │
│  SYNC FLOW:                                                  │
│                                                              │
│  1. HANDSHAKE                                                │
│     A → B: {type: "hello", vendor: "ABC", batch: 123,       │
│             checkpoint: "abc123"}                            │
│     B → A: {type: "hello", vendor: "ABC", batch: 123,       │
│             checkpoint: "def456"}                            │
│                                                              │
│  2. DELTA EXCHANGE                                           │
│     A → B: {type: "ops", data: [CRDT ops since B's point]}  │
│     B → A: {type: "ops", data: [CRDT ops since A's point]}  │
│                                                              │
│  3. CONTINUOUS SYNC                                          │
│     A → B: {type: "op", table: "bc_barcode",                │
│             operation: "UPDATE", row: {...}}                 │
│     B → A: {type: "ack", id: 123}                           │
│                                                              │
│  4. CONFLICT RESOLUTION                                      │
│     Same CRDT rules as server sync                          │
│     LWW for most fields                                     │
│     First-write-wins for pairing                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 Mesh Network

```
┌─────────────────────────────────────────────────────────────┐
│              MESH TOPOLOGY                                   │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  STAR (Simple):                MESH (Robust):                │
│                                                              │
│       B     C                   B ──── C                     │
│        \   /                    │\    /│                     │
│         \ /                     │ \  / │                     │
│          A (hub)                │  \/  │                     │
│         / \                     │  /\  │                     │
│        /   \                    │ /  \ │                     │
│       D     E                   D ──── E                     │
│                                                              │
│  Hub advantages:              Mesh advantages:               │
│  • Simple sync logic          • No single point of failure  │
│  • Lower bandwidth            • Any device can leave        │
│  • Easy to manage             • More resilient              │
│                                                              │
│  RECOMMENDATION: Star for < 5 devices, Mesh for > 5         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Implementation Options

### 4.1 Technology Stack

| Component | Option 1 | Option 2 | Recommendation |
|-----------|----------|----------|----------------|
| Discovery | mDNS | QR Code | mDNS + QR fallback |
| Transport | WebSocket | WebRTC | WebSocket |
| Protocol | Custom CRDT | Y.js | Y.js (battle-tested) |
| Encryption | TLS | DTLS | TLS |

### 4.2 Y.js Based Implementation

```typescript
// p2p-sync.ts
import * as Y from 'yjs'
import { WebsocketProvider } from 'y-websocket'
import { IndexeddbPersistence } from 'y-indexeddb'

// Shared document for sync
const ydoc = new Y.Doc()

// Local persistence
const persistence = new IndexeddbPersistence('barcode-sync', ydoc)

// P2P connection (when peer discovered)
function connectToPeer(peerUrl: string) {
  const wsProvider = new WebsocketProvider(peerUrl, 'barcode-room', ydoc)
  
  wsProvider.on('sync', (isSynced: boolean) => {
    if (isSynced) {
      console.log('Synced with peer!')
    }
  })
  
  return wsProvider
}

// Shared data structures
const barcodes = ydoc.getMap('barcodes')
const parents = ydoc.getMap('parents')
const pairings = ydoc.getMap('pairings')

// Add barcode
function addBarcode(id: string, data: any) {
  barcodes.set(id, data)
}

// Pair barcode to parent (first-write-wins)
function pairBarcode(barcodeId: string, parentId: string) {
  const existing = pairings.get(barcodeId)
  if (!existing) {
    pairings.set(barcodeId, {
      parentId,
      pairedAt: Date.now(),
      pairedBy: getCurrentUserId()
    })
  }
}

// Listen for changes from peers
barcodes.observe(event => {
  event.changes.keys.forEach((change, key) => {
    if (change.action === 'add') {
      console.log(`New barcode from peer: ${key}`)
      updateUI()
    }
  })
})
```

### 4.3 WebRTC Alternative

```typescript
// For direct browser-to-browser (no local server needed)
import { WebrtcProvider } from 'y-webrtc'

const provider = new WebrtcProvider('barcode-room', ydoc, {
  signaling: ['wss://signaling.barcode-app.com'],  // Fallback
  // For LAN-only, use local signaling or ICE candidates
})
```

---

## 5. Security Considerations

### 5.1 P2P Security Model

```
┌─────────────────────────────────────────────────────────────┐
│              P2P SECURITY                                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  AUTHENTICATION:                                             │
│  • Same JWT token required for P2P                          │
│  • Validate vendor_code matches                             │
│  • Validate batch assignment                                │
│                                                              │
│  AUTHORIZATION:                                              │
│  • Only sync data for shared batches                        │
│  • Cannot access other vendor's data                        │
│                                                              │
│  ENCRYPTION:                                                 │
│  • TLS for all P2P connections                              │
│  • Certificate pinning between known devices                │
│                                                              │
│  TRUST MODEL:                                                │
│  • Device must be registered with server first              │
│  • P2P only allowed for known device pairs                  │
│  • Server can revoke P2P capability remotely                │
│                                                              │
│  RISK MITIGATION:                                            │
│  • Audit log of all P2P syncs                               │
│  • Anomaly detection (unusual sync volume)                  │
│  • Time-limited P2P sessions (auto-disconnect)              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Hybrid Sync Flow

```
┌─────────────────────────────────────────────────────────────┐
│              HYBRID SYNC DECISION TREE                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  START                                                       │
│    │                                                         │
│    ▼                                                         │
│  ┌──────────────────┐                                       │
│  │ Internet         │                                       │
│  │ available?       │                                       │
│  └────────┬─────────┘                                       │
│           │                                                  │
│     YES   │   NO                                             │
│     ▼     │   ▼                                              │
│  ┌────────┴───┐  ┌────────────┐                             │
│  │ Sync via   │  │ Peers on   │                             │
│  │ Electric   │  │ same LAN?  │                             │
│  │ (primary)  │  └─────┬──────┘                             │
│  └────────────┘        │                                     │
│                   YES  │   NO                                │
│                   ▼    │   ▼                                 │
│            ┌──────┴────┐  ┌────────────┐                    │
│            │ P2P sync  │  │ Queue all  │                    │
│            │ via LAN   │  │ operations │                    │
│            └───────────┘  │ locally    │                    │
│                           └────────────┘                    │
│                                                              │
│  PRIORITY ORDER:                                             │
│  1. Internet → Electric (authoritative)                     │
│  2. LAN → P2P sync (temporary, merge later)                 │
│  3. Offline → Local queue (sync when possible)              │
│                                                              │
│  MERGE STRATEGY:                                             │
│  When internet returns:                                      │
│  1. P2P changes treated as "local changes"                  │
│  2. All merge to server via Electric                        │
│  3. CRDT handles any conflicts                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. UI/UX for P2P

### 7.1 P2P Status Indicator

```
┌──────────────────────────────────────────┐
│ ☰  Barcode Scanner                       │
├──────────────────────────────────────────┤
│                                          │
│  Sync Status:                            │
│  ┌────────────────────────────────────┐  │
│  │ 📶 Server: Offline                  │  │
│  │ 🔗 P2P: 2 devices connected         │  │
│  │    • Device-B (Dewi)                │  │
│  │    • Device-C (Budi)                │  │
│  └────────────────────────────────────┘  │
│                                          │
│  [Disconnect P2P] [Show QR to Connect]   │
│                                          │
└──────────────────────────────────────────┘
```

### 7.2 Connection Flow

```
Step 1: Discover
┌────────────────────────────┐
│ Looking for nearby devices │
│ 🔍 Scanning...             │
│                            │
│ Found:                     │
│ ○ Device-B (Dewi)          │
│ ○ Device-C (Budi)          │
│                            │
│ [Connect All] [Select]     │
└────────────────────────────┘

Step 2: Confirm
┌────────────────────────────┐
│ Connect to Device-B?       │
│                            │
│ Vendor: ABC                │
│ Batch: POC-001             │
│ User: Dewi                 │
│                            │
│ [Cancel] [Connect]         │
└────────────────────────────┘

Step 3: Connected
┌────────────────────────────┐
│ ✓ Connected to Device-B    │
│                            │
│ Syncing 150 items...       │
│ ████████░░░░░░ 60%         │
│                            │
└────────────────────────────┘
```

---

## 8. Implementation Phases

| Phase | Scope | Duration |
|-------|-------|----------|
| Phase 1 | QR-based manual connection | Week 1 |
| Phase 2 | mDNS auto-discovery | Week 2 |
| Phase 3 | Y.js CRDT sync | Week 3 |
| Phase 4 | Hybrid Electric + P2P | Week 4 |
| Phase 5 | Testing & hardening | Week 5 |

---

## 9. Pros & Cons

### Advantages

- ✅ Works without internet
- ✅ Lower latency (LAN is faster)
- ✅ Reduces server load
- ✅ Better UX for field operations

### Disadvantages

- ❌ More complex implementation
- ❌ Security requires careful design
- ❌ Debugging harder with multiple sync paths
- ❌ Merge conflicts possible between P2P and server

---

## 10. Recommendation

```
┌─────────────────────────────────────────────────────────────┐
│                    RECOMMENDATION                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PHASE 1 (MVP): Server-only sync (ElectricSQL)              │
│  • Get basic local-first working first                      │
│  • Validate CRDT approach                                   │
│                                                              │
│  PHASE 2 (v1.1): Add P2P as enhancement                     │
│  • After MVP stable, add P2P capability                     │
│  • Start with QR-based manual connection                    │
│  • Use Y.js for proven CRDT implementation                  │
│                                                              │
│  PHASE 3 (v1.2): Auto-discovery                             │
│  • mDNS for seamless device finding                         │
│  • Mesh network for larger teams                            │
│                                                              │
│  This staged approach:                                       │
│  • Reduces initial complexity                               │
│  • Allows learning from production usage                    │
│  • P2P builds on proven foundation                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

