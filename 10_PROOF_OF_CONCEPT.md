# Proof of Concept
## ElectricSQL Local-First Sync Demo

---

## 1. POC Objectives

```
┌─────────────────────────────────────────────────────────────┐
│                    POC OBJECTIVES                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Validate ElectricSQL with PostgreSQL                    │
│  2. Demonstrate real-time sync across devices               │
│  3. Test offline capability                                 │
│  4. Measure sync performance                                │
│  5. Identify integration challenges                         │
│                                                              │
│  SUCCESS CRITERIA:                                           │
│  ✓ Sync 10,000 barcodes in < 60 seconds                     │
│  ✓ Offline writes sync correctly on reconnect               │
│  ✓ Conflicts resolved automatically                         │
│  ✓ No data loss in any scenario                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. POC Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    POC ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐   │
│  │   Device A  │     │   Device B  │     │   Device C  │   │
│  │  (Mobile)   │     │  (Tablet)   │     │   (Web)     │   │
│  │  SQLite     │     │  SQLite     │     │  IndexedDB  │   │
│  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘   │
│         │                   │                   │           │
│         └───────────────────┼───────────────────┘           │
│                             │                               │
│                     ┌───────┴───────┐                       │
│                     │  Electric     │                       │
│                     │  (Docker)     │                       │
│                     └───────┬───────┘                       │
│                             │                               │
│                     ┌───────┴───────┐                       │
│                     │  PostgreSQL   │                       │
│                     │  (Docker)     │                       │
│                     └───────────────┘                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Setup Instructions

### 3.1 Docker Compose

```yaml
# docker-compose.poc.yaml
version: '3.8'

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: barcode_poc
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    command: 
      - postgres
      - -c
      - wal_level=logical
    ports:
      - "5432:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

  electric:
    image: electricsql/electric:latest
    environment:
      DATABASE_URL: postgresql://postgres:postgres@postgres:5432/barcode_poc
      AUTH_MODE: insecure  # For POC only
      ELECTRIC_WRITE_TO_PG_MODE: direct_writes
    ports:
      - "5133:5133"
    depends_on:
      - postgres

volumes:
  pg_data:
```

### 3.2 Database Schema

```sql
-- init.sql
CREATE SCHEMA IF NOT EXISTS app_barcode;

CREATE TABLE app_barcode.bc_batch (
    id SERIAL PRIMARY KEY,
    vendor_code VARCHAR(100) NOT NULL,
    name VARCHAR(300) NOT NULL,
    is_active BOOLEAN DEFAULT FALSE,
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE app_barcode.bc_parent (
    id SERIAL PRIMARY KEY,
    vendor_code VARCHAR(100) NOT NULL,
    batch_id INTEGER REFERENCES app_barcode.bc_batch(id),
    code VARCHAR(200) NOT NULL,
    qty INTEGER DEFAULT 0,
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE app_barcode.bc_barcode (
    id SERIAL PRIMARY KEY,
    vendor_code VARCHAR(100) NOT NULL,
    batch_id INTEGER REFERENCES app_barcode.bc_batch(id),
    parent_id INTEGER REFERENCES app_barcode.bc_parent(id),
    barcode VARCHAR(200) UNIQUE NOT NULL,
    note TEXT,
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Seed data
INSERT INTO app_barcode.bc_batch (vendor_code, name, is_active) 
VALUES ('VENDOR_A', 'POC Batch 1', true);

INSERT INTO app_barcode.bc_parent (vendor_code, batch_id, code, qty)
SELECT 'VENDOR_A', 1, 'PARENT-' || i, 10
FROM generate_series(1, 100) AS i;

INSERT INTO app_barcode.bc_barcode (vendor_code, batch_id, barcode)
SELECT 'VENDOR_A', 1, 'BC-' || LPAD(i::text, 6, '0')
FROM generate_series(1, 10000) AS i;
```

### 3.3 Client Setup (React)

```bash
# Create POC client
npx create-vite@latest poc-client --template react-ts
cd poc-client

# Install dependencies
npm install @electric-sql/client better-sqlite3
npm install -D @electric-sql/cli
```

### 3.4 Client Code

```typescript
// src/electric.ts
import { ElectricClient, ShapeStream } from '@electric-sql/client'

const ELECTRIC_URL = 'http://localhost:5133'

export async function createShape(batchId: number) {
  const stream = new ShapeStream({
    url: `${ELECTRIC_URL}/v1/shape`,
    params: {
      table: 'app_barcode.bc_barcode',
      where: `batch_id = ${batchId} AND is_deleted = false`
    }
  })
  
  return stream
}

// src/App.tsx
import { useState, useEffect } from 'react'
import { createShape } from './electric'

function App() {
  const [barcodes, setBarcodes] = useState([])
  const [syncStatus, setSyncStatus] = useState('connecting')
  const [lastSync, setLastSync] = useState(null)

  useEffect(() => {
    const initSync = async () => {
      const shape = await createShape(1)
      
      shape.subscribe((messages) => {
        // Apply changes to local state
        messages.forEach(msg => {
          if (msg.headers.operation === 'insert') {
            setBarcodes(prev => [...prev, msg.value])
          } else if (msg.headers.operation === 'update') {
            setBarcodes(prev => 
              prev.map(b => b.id === msg.value.id ? msg.value : b)
            )
          }
        })
        
        setSyncStatus('synced')
        setLastSync(new Date())
      })
    }
    
    initSync()
  }, [])

  return (
    <div>
      <h1>ElectricSQL POC</h1>
      <p>Status: {syncStatus}</p>
      <p>Last sync: {lastSync?.toLocaleTimeString()}</p>
      <p>Barcodes: {barcodes.length}</p>
      
      <ul>
        {barcodes.slice(0, 20).map(b => (
          <li key={b.id}>{b.barcode}</li>
        ))}
      </ul>
    </div>
  )
}
```

---

## 4. Test Scenarios

### 4.1 Initial Sync Test

```
SCENARIO: First device sync
GIVEN: 10,000 barcodes in PostgreSQL
WHEN: Client connects to Electric
THEN: All 10,000 barcodes sync to client
EXPECTED: < 60 seconds
```

### 4.2 Real-time Sync Test

```
SCENARIO: Insert on one device, see on another
GIVEN: Two devices connected
WHEN: Device A inserts new barcode via PostgreSQL
THEN: Device B sees new barcode within 1 second
```

### 4.3 Offline Write Test

```
SCENARIO: Write while offline
GIVEN: Device connected and synced
WHEN: 
  1. Disconnect network
  2. Update barcode note
  3. Reconnect network
THEN: 
  1. Local update succeeds immediately
  2. Update syncs to server on reconnect
```

### 4.4 Conflict Test

```
SCENARIO: Concurrent edits
GIVEN: Two devices with same barcode
WHEN:
  1. Both devices go offline
  2. Device A: note = "QC OK"
  3. Device B: note = "Shipped"
  4. Both reconnect
THEN:
  1. LWW resolves conflict (latest timestamp wins)
  2. Both devices show same final value
```

---

## 5. Performance Benchmarks

### 5.1 Sync Speed

| Data Volume | Expected Time | Actual Time | Status |
|-------------|---------------|-------------|--------|
| 1,000 rows | < 5 sec | TBD | ⏳ |
| 10,000 rows | < 60 sec | TBD | ⏳ |
| 100,000 rows | < 5 min | TBD | ⏳ |

### 5.2 Latency

| Operation | Expected | Actual | Status |
|-----------|----------|--------|--------|
| Local write | < 10ms | TBD | ⏳ |
| Sync propagation | < 500ms | TBD | ⏳ |
| Reconnection | < 3 sec | TBD | ⏳ |

---

## 6. Run POC

```bash
# 1. Start services
docker-compose -f docker-compose.poc.yaml up -d

# 2. Wait for Electric to connect
docker logs -f electric

# 3. Start client
cd poc-client
npm run dev

# 4. Open browser
open http://localhost:5173
```

---

## 7. POC Findings Template

```markdown
## POC Results

### Date: ____

### Success Criteria

| Criteria | Met | Notes |
|----------|-----|-------|
| Sync 10K in < 60s | ☐ | |
| Offline writes work | ☐ | |
| Conflicts resolved | ☐ | |
| No data loss | ☐ | |

### Performance Results

Initial sync (10K rows): ____ seconds
Real-time propagation: ____ ms
Reconnection time: ____ seconds

### Issues Found

1. ____
2. ____

### Recommendations

1. ____
2. ____

### Go/No-Go Decision

[ ] GO - Proceed to Phase 1
[ ] NO-GO - Reason: ____
```

