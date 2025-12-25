# ElectricSQL Sync - Versi Minimal
## Local-First Sync Paling Simpel

---

## TL;DR - Apa yang Kita Butuhkan?

```
┌─────────────────────────────────────────────────────────────┐
│                    MINIMAL SYNC                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  CLIENT (Mobile/Web)                                         │
│  ┌─────────────────────┐                                    │
│  │  SQLite / IndexedDB │ ← Semua data lokal                 │
│  │  + Offline Queue    │ ← Simpan perubahan saat offline    │
│  └──────────┬──────────┘                                    │
│             │                                                │
│             │ HTTP/WebSocket                                │
│             │                                                │
│  SERVER                                                      │
│  ┌──────────┴──────────┐                                    │
│  │     ElectricSQL     │ ← Sync engine                      │
│  │     (Docker)        │                                    │
│  └──────────┬──────────┘                                    │
│             │                                                │
│  ┌──────────┴──────────┐                                    │
│  │     PostgreSQL      │ ← Database utama                   │
│  └─────────────────────┘                                    │
│                                                              │
│  ITU SAJA. TIDAK PERLU LEBIH.                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. Setup Server (15 menit)

### docker-compose.yml

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: barcode
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret123
    command: postgres -c wal_level=logical
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  electric:
    image: electricsql/electric:latest
    environment:
      DATABASE_URL: postgresql://postgres:secret123@postgres:5432/barcode
      AUTH_MODE: insecure  # Untuk development
    ports:
      - "5133:5133"
    depends_on:
      - postgres

volumes:
  pgdata:
```

### Jalankan

```bash
docker-compose up -d
```

**Selesai.** Server sync sudah jalan.

---

## 2. Setup Client (30 menit)

### Install Dependencies

```bash
npm install @electric-sql/client
# atau untuk React Native
npm install @electric-sql/react
```

### Kode Minimal

```typescript
// sync.ts
import { ShapeStream, Shape } from '@electric-sql/client'

const ELECTRIC_URL = 'http://localhost:5133'

// 1. Subscribe ke data
const stream = new ShapeStream({
  url: `${ELECTRIC_URL}/v1/shape`,
  params: {
    table: 'bc_barcode',
    where: `batch_id = 1`  // Filter data yang dibutuhkan
  }
})

// 2. Simpan ke local storage
const shape = new Shape(stream)

shape.subscribe(({ rows }) => {
  // rows = data terbaru dari server
  // Simpan ke SQLite/IndexedDB
  localStorage.setItem('barcodes', JSON.stringify(rows))
  
  // Update UI
  updateUI(rows)
})

// 3. Kirim perubahan lokal ke server
async function saveBarcode(data) {
  // Simpan lokal dulu (instant)
  const local = JSON.parse(localStorage.getItem('barcodes') || '[]')
  local.push(data)
  localStorage.setItem('barcodes', JSON.stringify(local))
  
  // Kirim ke server (background)
  try {
    await fetch('/api/barcode', {
      method: 'POST',
      body: JSON.stringify(data)
    })
  } catch (e) {
    // Offline? Simpan ke queue
    const queue = JSON.parse(localStorage.getItem('pendingQueue') || '[]')
    queue.push(data)
    localStorage.setItem('pendingQueue', JSON.stringify(queue))
  }
}

// 4. Process offline queue saat online
window.addEventListener('online', async () => {
  const queue = JSON.parse(localStorage.getItem('pendingQueue') || '[]')
  for (const item of queue) {
    await fetch('/api/barcode', {
      method: 'POST',
      body: JSON.stringify(item)
    })
  }
  localStorage.setItem('pendingQueue', '[]')
})
```

---

## 3. Cara Kerja (Super Simpel)

```
ONLINE:
─────────
User scan → Simpan lokal → Kirim server → Server broadcast ke device lain
                ↑                                      ↓
                └──────────────────────────────────────┘
                          (Real-time sync)

OFFLINE:
────────
User scan → Simpan lokal → Tambah ke queue
                              ↓
                       [Tunggu online]
                              ↓
                       Kirim semua queue
```

---

## 4. Conflict Resolution (Simpel)

```
ATURAN: LAST WRITE WINS

Device A: update barcode.note = "OK" jam 10:00
Device B: update barcode.note = "NG" jam 10:01

Hasil: barcode.note = "NG" (yang terakhir menang)

SUDAH. Tidak perlu CRDT kompleks untuk MVP.
```

---

## 5. Yang TIDAK Perlu di MVP

| Fitur | Status | Alasan |
|-------|--------|--------|
| CRDT | ❌ Skip | Last-write-wins cukup untuk MVP |
| P2P Sync | ❌ Skip | Tambah nanti kalau butuh |
| SQLCipher | ❌ Skip | Enkripsi nanti di production |
| Mesh Network | ❌ Skip | Overkill |
| Multi-region | ❌ Skip | 1 server cukup dulu |
| Monitoring | ❌ Skip | Console.log dulu |

---

## 6. Minimal API (FastAPI)

```python
# main.py
from fastapi import FastAPI
from sqlalchemy import create_engine, text

app = FastAPI()
engine = create_engine("postgresql://postgres:secret123@localhost:5432/barcode")

@app.post("/api/barcode")
async def create_barcode(data: dict):
    with engine.connect() as conn:
        conn.execute(text("""
            INSERT INTO bc_barcode (barcode, batch_id, vendor_code)
            VALUES (:barcode, :batch_id, :vendor_code)
        """), data)
        conn.commit()
    return {"status": "ok"}

@app.put("/api/barcode/{id}")
async def update_barcode(id: int, data: dict):
    with engine.connect() as conn:
        conn.execute(text("""
            UPDATE bc_barcode SET note = :note, updated_at = now()
            WHERE id = :id
        """), {"id": id, **data})
        conn.commit()
    return {"status": "ok"}
```

---

## 7. Timeline MVP

| Hari | Task |
|------|------|
| Hari 1 | Setup Docker + Electric + PostgreSQL |
| Hari 2 | Client basic sync (read-only) |
| Hari 3 | Client write + offline queue |
| Hari 4 | Testing manual |
| Hari 5 | Deploy ke staging |

**Total: 1 minggu untuk MVP yang bisa offline.**

---

## 8. Upgrade Path

Kalau MVP sudah jalan dan butuh fitur lebih:

```
MVP (Minggu 1-2)
    │
    ├── + Auth (JWT) ──────────────── Minggu 3
    │
    ├── + SQLite proper ───────────── Minggu 4
    │
    ├── + Conflict detection ──────── Minggu 5
    │
    ├── + P2P sync ────────────────── Minggu 6+
    │
    └── + Production hardening ────── Minggu 7+
```

---

## 9. Kesimpulan

```
┌─────────────────────────────────────────────────────────────┐
│                    MINIMAL VIABLE SYNC                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  YANG DIBUTUHKAN:                                            │
│  ✅ PostgreSQL (sudah ada)                                  │
│  ✅ ElectricSQL (1 container Docker)                        │
│  ✅ Client library (@electric-sql/client)                   │
│  ✅ LocalStorage/IndexedDB untuk offline                    │
│  ✅ Simple queue untuk pending changes                      │
│                                                              │
│  YANG TIDAK DIBUTUHKAN (MVP):                               │
│  ❌ CRDT kompleks                                           │
│  ❌ P2P networking                                          │
│  ❌ Multi-region                                            │
│  ❌ Enkripsi client-side                                    │
│  ❌ Custom conflict resolution                              │
│                                                              │
│  HASIL:                                                      │
│  • Scan barcode works offline ✓                             │
│  • Auto-sync saat online ✓                                  │
│  • Real-time update antar device ✓                          │
│  • Implementasi: 1 minggu ✓                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 10. File Structure MVP

```
project/
├── docker-compose.yml      # Electric + PostgreSQL
├── backend/
│   └── main.py             # FastAPI (10 lines)
└── client/
    └── sync.ts             # Sync logic (50 lines)
```

**Itu saja. Mulai dari sini, tambah complexity sesuai kebutuhan.**

