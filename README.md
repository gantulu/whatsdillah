# WhatsApp Cloud API Integration
## Technical Specification — v1.4

**Version:** 1.4  
**Status:** Draft  
**Architecture:** Supabase Edge Functions + PostgreSQL + Realtime

---

## 1. Project Overview

Membangun aplikasi WhatsApp berbasis WhatsApp Cloud API resmi Meta dengan kemampuan:

1. Menerima pesan teks pelanggan.
2. Menampilkan percakapan secara realtime.
3. Membalas pesan teks pelanggan.
4. Menyimpan kontak dan riwayat pesan.
5. Memperbarui status pesan secara realtime.

### Prinsip arsitektur

- Tidak menggunakan Node.js + Express.
- Backend menggunakan Supabase Edge Functions.
- Runtime Edge Functions menggunakan Deno.
- Database menggunakan Supabase PostgreSQL.
- Sinkronisasi realtime menggunakan Supabase Realtime.
- Frontend menggunakan Supabase JavaScript Client.
- Tidak menggunakan Supabase Auth.
- Database menggunakan public access.
- Konfigurasi Supabase URL dan Publishable Key hardcoded di frontend.

## 2. Technology Stack

| Layer | Technology |
|---|---|
| Frontend | React |
| Build Tool | Vite |
| Language | TypeScript |
| UI | WhatsApp Mobile Design System |
| Database | Supabase PostgreSQL |
| Database Client | @supabase/supabase-js |
| Backend | Supabase Edge Functions |
| Runtime | Deno |
| Realtime | Supabase Realtime |
| WhatsApp API | Meta WhatsApp Cloud API |
| Frontend Hosting | GitHub Pages |
| Backend Hosting | Supabase |

## 3. System Architecture

```text
                 WHATSAPP CUSTOMER
                         |
                         v
                META CLOUD API
                         |
                 +-------+-------+
                 |               |
                 v               ^
          Incoming Webhook    Send Message
                 |               |
                 v               |
        SUPABASE EDGE FUNCTIONS
                 |
        +--------+---------+
        |                  |
        v                  v
   PostgreSQL          Meta Graph API
        |
        v
   SUPABASE REALTIME
        |
        v
   REACT FRONTEND
        |
        v
   WHATSAPP MOBILE UI
```

### Component responsibilities

| Component | Responsibility |
|---|---|
| React | Antarmuka aplikasi |
| Supabase Client | Operasi database dari frontend |
| Edge Functions | Backend dan integrasi Meta |
| PostgreSQL | Penyimpanan kontak dan pesan |
| Realtime | Sinkronisasi perubahan data |
| Meta Cloud API | Pengiriman dan penerimaan pesan |

## 4. Supabase Client

Installation:

```bash
npm install @supabase/supabase-js
```

File: `frontend/src/lib/supabase.ts`

```typescript
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = 'https://YOUR_PROJECT_ID.supabase.co'

const supabasePublishableKey = 'sb_publishable_YOUR_KEY'

export const supabase = createClient(
  supabaseUrl,
  supabasePublishableKey
)
```

Ketentuan:

- Supabase URL hardcoded.
- Publishable Key hardcoded.
- Tidak menggunakan Supabase Auth.
- Tidak memasukkan Meta Access Token ke frontend.
- Tidak memasukkan Supabase Secret Key ke frontend.

## 5. Database Design

Versi ini menggunakan dua tabel utama.

### 5.1 contacts

| Column | Type | Description |
|---|---|---|
| id | UUID | Primary Key |
| phone_number | TEXT | Nomor WhatsApp |
| name | TEXT | Nama kontak |
| last_message_at | TIMESTAMPTZ | Pesan terakhir |
| created_at | TIMESTAMPTZ | Waktu dibuat |
| updated_at | TIMESTAMPTZ | Waktu diperbarui |

### 5.2 messages

| Column | Type | Description |
|---|---|---|
| id | UUID | Primary Key |
| contact_id | UUID | Foreign Key |
| wamid | TEXT | Meta Message ID |
| direction | TEXT | incoming / outgoing |
| message_type | TEXT | text |
| message | TEXT | Isi pesan |
| status | TEXT | Status pengiriman |
| created_at | TIMESTAMPTZ | Waktu dibuat |
| updated_at | TIMESTAMPTZ | Waktu diperbarui |

Constraints:

- `contacts.phone_number` harus unique.
- `messages.wamid` harus unique jika tersedia.
- `messages.contact_id` mengacu ke `contacts.id`.
- `direction` hanya menerima `incoming` atau `outgoing`.
- `message_type` untuk v1.4 hanya `text`.

### 5.3 Database SQL

```sql
CREATE TABLE public.contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone_number TEXT NOT NULL UNIQUE,
    name TEXT,
    last_message_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id UUID NOT NULL
        REFERENCES public.contacts(id)
        ON DELETE CASCADE,
    wamid TEXT UNIQUE,
    direction TEXT NOT NULL
        CHECK (direction IN ('incoming', 'outgoing')),
    message_type TEXT NOT NULL DEFAULT 'text'
        CHECK (message_type = 'text'),
    message TEXT NOT NULL,
    status TEXT DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_messages_contact_id
ON public.messages(contact_id);

CREATE INDEX idx_messages_created_at
ON public.messages(created_at DESC);
```

## 6. Public Database Access

Tidak menggunakan Supabase Auth maupun login pengguna.

### Configuration

| Setting | Value |
|---|---|
| Supabase Auth | Not Used |
| Login | Not Required |
| Database Access | Public |
| Publishable Key | Frontend |
| RLS | Enabled |
| Public Policies | Enabled |
| Additional User Authentication | None |

### Public Policies

```sql
ALTER TABLE public.contacts ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.messages ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Public access contacts"
ON public.contacts
FOR ALL
TO anon
USING (true)
WITH CHECK (true);

CREATE POLICY "Public access messages"
ON public.messages
FOR ALL
TO anon
USING (true)
WITH CHECK (true);
```

Konsekuensi: siapa pun yang memiliki project URL dan Publishable Key dapat membaca atau mengubah data yang diizinkan oleh policies tersebut.

## 7. Supabase Edge Functions

Edge Functions menjadi satu-satunya backend aplikasi.

### 7.1 Project Structure

```text
supabase/
└── functions/
    ├── whatsapp-webhook/
    │   └── index.ts
    │
    ├── whatsapp-send-message/
    │   └── index.ts
    │
    └── whatsapp-webhook-verify/
        └── index.ts
```

### 7.2 Function Responsibilities

| Function | Responsibility |
|---|---|
| whatsapp-webhook | Menerima pesan dan status dari Meta |
| whatsapp-send-message | Mengirim pesan ke pelanggan |
| whatsapp-webhook-verify | Menangani verifikasi webhook Meta |

Semua function menggunakan TypeScript dengan Deno runtime.

### 7.3 Webhook Endpoint

```http
GET  /functions/v1/whatsapp-webhook
POST /functions/v1/whatsapp-webhook
```

GET menangani webhook verification challenge dari Meta.

POST menerima event pesan masuk dan status pesan.

### 7.4 Send Message Endpoint

```http
POST /functions/v1/whatsapp-send-message
```

Request:

```json
{
  "contact_id": "contact-001",
  "message": "Halo! Produk masih tersedia."
}
```

Response:

```json
{
  "success": true,
  "data": {
    "message_id": "wamid.example",
    "status": "sent"
  }
}
```

Endpoint pengiriman pesan dipanggil dari frontend menggunakan Supabase Client.

## 8. Receive Messages

### Incoming Message Flow

```text
Customer sends message
          |
          v
Meta WhatsApp Cloud API
          |
          v
whatsapp-webhook
          |
          v
Parse incoming event
          |
          v
Find or create contact
          |
          v
Insert message
          |
          v
Update last_message_at
          |
          v
Supabase Realtime
          |
          v
React Chat UI
```

### Requirements

- Menerima pesan melalui webhook Meta.
- Memproses pesan teks.
- Membuat kontak otomatis jika belum ada.
- Menyimpan pesan ke PostgreSQL.
- Mencegah duplikasi berdasarkan WAMID.
- Memperbarui waktu pesan terakhir.
- Menampilkan pesan baru melalui Realtime.

## 9. Reply Messages

### Meta Graph API

```http
POST https://graph.facebook.com/{VERSION}/{PHONE_NUMBER_ID}/messages
```

### Outgoing Flow

```text
Admin types message
         |
         v
React Frontend
         |
         v
whatsapp-send-message
         |
         v
Meta Graph API
         |
         v
Meta API Response
         |
         v
Save message and WAMID
         |
         v
Supabase Realtime
         |
         v
Update Chat UI
```

### Meta Request

```json
{
  "messaging_product": "whatsapp",
  "recipient_type": "individual",
  "to": "6281234567890",
  "type": "text",
  "text": {
    "preview_url": false,
    "body": "Halo! Produk masih tersedia."
  }
}
```

### Requirements

- Pesan tidak boleh kosong.
- Penerima menggunakan nomor internasional.
- Meta Access Token hanya berada di Edge Function.
- Simpan WAMID dari respons Meta.
- Simpan status pengiriman.
- Tampilkan error jika Meta API gagal.
- Patuhi customer service window dan ketentuan template Meta.

## 10. Supabase Realtime

Realtime bertugas menyampaikan perubahan database kepada frontend melalui WebSocket.

### 10.1 Enable Realtime

```sql
ALTER PUBLICATION supabase_realtime
ADD TABLE public.messages;
```

### 10.2 Frontend Subscription

```typescript
import { supabase } from '@/lib/supabase'

const channel = supabase
  .channel('whatsapp-messages')
  .on(
    'postgres_changes',
    {
      event: '*',
      schema: 'public',
      table: 'messages'
    },
    (payload) => {
      console.log('Message update:', payload)
    }
  )
  .subscribe()
```

### 10.3 Realtime Events

| Event | Usage |
|---|---|
| INSERT | Pesan baru |
| UPDATE | Perubahan status pesan |
| DELETE | Penghapusan pesan |

### 10.4 Realtime Requirements

- Pesan masuk langsung muncul.
- Pesan keluar dapat disinkronkan.
- Status sent diperbarui.
- Status delivered diperbarui.
- Status read diperbarui.
- UI tidak memerlukan refresh.
- Hindari duplikasi pesan.
- Unsubscribe ketika component dilepas.

Realtime hanya menyinkronkan perubahan database. Pengiriman pesan WhatsApp tetap dilakukan melalui Edge Function dan Meta Graph API.

## 11. Message Status

Status pesan yang didukung:

| Status | Description |
|---|---|
| pending | Menunggu pengiriman |
| sent | Diterima Meta untuk pengiriman |
| delivered | Sampai ke perangkat penerima |
| read | Dibaca penerima |
| failed | Pengiriman gagal |

Status diperbarui berdasarkan webhook status dari Meta.

Incoming messages juga dapat memiliki status internal yang sesuai kebutuhan aplikasi.

## 12. Environment Secrets

Simpan credential di Supabase Edge Function Secrets.

```env
META_ACCESS_TOKEN=
META_APP_SECRET=
META_VERIFY_TOKEN=

WHATSAPP_PHONE_NUMBER_ID=
WHATSAPP_BUSINESS_ACCOUNT_ID=

META_GRAPH_API_VERSION=
```

Backend juga menggunakan credential Supabase server-side yang diperlukan untuk operasi database.

Jangan memasukkan credential rahasia ke frontend atau repository GitHub.

## 13. Webhook Requirements

- GET verification harus mengembalikan challenge sesuai permintaan Meta.
- POST webhook harus memproses payload Meta.
- Validasi `X-Hub-Signature-256` menggunakan Meta App Secret.
- Signature diverifikasi terhadap raw request body.
- Gunakan idempotency berdasarkan WAMID.
- Respons webhook diberikan segera setelah event diterima dan diproses sesuai kebutuhan.
- Tangani retry dari Meta tanpa membuat pesan duplikat.

Webhook Meta merupakan endpoint eksternal. Konfigurasi function harus memungkinkan request Meta masuk tanpa Supabase JWT, lalu melakukan verifikasi signature Meta di dalam handler.

## 14. Frontend UI

### Main Screens

| Screen | Features |
|---|---|
| Chat List | Daftar percakapan |
| Conversation | Riwayat pesan |
| Message Input | Menulis balasan |
| Contact Information | Informasi kontak |

### Chat List

- Nama kontak.
- Nomor WhatsApp.
- Pesan terakhir.
- Waktu pesan terakhir.
- Indikator pesan belum dibaca.

### Conversation

- Incoming message bubbles.
- Outgoing message bubbles.
- Timestamp.
- Message status.
- Text input.
- Send button.

Desain menggunakan pendekatan WhatsApp Mobile UI dan dioptimalkan untuk tampilan mobile.

## 15. Project Structure

```text
whatsapp/
│
├── frontend/
│   └── src/
│       ├── lib/
│       │   └── supabase.ts
│       ├── components/
│       ├── pages/
│       ├── features/
│       │   └── chat/
│       │       ├── components/
│       │       ├── hooks/
│       │       └── services/
│       ├── services/
│       └── types/
│
├── supabase/
│   ├── config.toml
│   ├── migrations/
│   └── functions/
│       ├── whatsapp-webhook/
│       │   └── index.ts
│       ├── whatsapp-send-message/
│       │   └── index.ts
│       └── whatsapp-webhook-verify/
│           └── index.ts
│
├── docs/
│   └── whatsapp-cloud-api/
│       ├── README.md
│       ├── v1.4-specification.md
│       ├── database.md
│       ├── webhook.md
│       ├── receive-messages.md
│       ├── reply-messages.md
│       └── realtime.md
│
└── README.md
```

## 16. Deployment

### Frontend

Deploy React + Vite ke GitHub Pages.

### Backend

Deploy seluruh Edge Functions ke Supabase.

### Database

Gunakan Supabase PostgreSQL.

### Realtime

Aktifkan Realtime pada tabel `messages`.

### Meta Webhook

Konfigurasikan callback URL:

```text
https://YOUR_PROJECT_REF.supabase.co/functions/v1/whatsapp-webhook
```

Masukkan Verify Token yang sama dengan secret `META_VERIFY_TOKEN`.

## 17. Acceptance Criteria

### Edge Functions
- [ ] Tidak ada Node.js + Express backend.
- [ ] Semua function berjalan dengan Deno.
- [ ] Webhook verification berhasil.
- [ ] Webhook receiver berhasil.
- [ ] Send message function berhasil.

### Receive Messages
- [ ] Pesan teks berhasil diterima.
- [ ] Kontak otomatis dibuat.
- [ ] Pesan tersimpan.
- [ ] Tidak ada duplikasi WAMID.

### Reply Messages
- [ ] Admin dapat mengirim balasan.
- [ ] Pesan dikirim melalui Meta Cloud API.
- [ ] WAMID tersimpan.
- [ ] Status pesan diperbarui.

### Realtime
- [ ] INSERT event berfungsi.
- [ ] UPDATE event berfungsi.
- [ ] Pesan baru langsung muncul.
- [ ] Status pesan langsung berubah.
- [ ] UI tidak memerlukan refresh.

### Supabase
- [ ] Supabase Client terhubung.
- [ ] Publishable Key digunakan di frontend.
- [ ] Public policies aktif.
- [ ] Tidak menggunakan Supabase Auth.

## 18. Development Roadmap

| Phase | Task |
|---|---|
| 1 | Supabase Project Setup |
| 2 | Database Schema |
| 3 | Public Access Policies |
| 4 | Edge Functions Setup |
| 5 | Meta Webhook Verification |
| 6 | Receive Text Messages |
| 7 | Supabase Realtime |
| 8 | Send Text Messages |
| 9 | Message Status Synchronization |
| 10 | React Frontend Integration |
| 11 | Testing |
| 12 | Deployment |

## 19. Official Documentation

- [Supabase Edge Functions](https://supabase.com/docs/guides/functions)
- [Supabase Edge Functions Limits](https://supabase.com/docs/guides/functions/limits)
- [Supabase Realtime](https://supabase.com/docs/guides/realtime)
- [Postgres Changes](https://supabase.com/docs/guides/realtime/postgres-changes)
- [Supabase API Keys](https://supabase.com/docs/guides/api/api-keys)
- [Supabase Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [Meta WhatsApp Cloud API](https://developers.facebook.com/docs/whatsapp/cloud-api/)
- [Meta Webhooks](https://developers.facebook.com/docs/whatsapp/cloud-api/webhooks/)
- [Meta Send Messages](https://developers.facebook.com/docs/whatsapp/cloud-api/guides/send-messages/)

---

## 20. Final Architecture Decision

| Component | Final Decision |
|---|---|
| Frontend | React + Vite + TypeScript |
| Backend | Supabase Edge Functions |
| Runtime | Deno |
| Database | Supabase PostgreSQL |
| Database Client | Supabase JS Client |
| Realtime | Supabase Realtime |
| WhatsApp API | Official Meta Cloud API |
| Authentication | None |
| Database Access | Public |
| RLS | Public policies |
| Frontend Hosting | GitHub Pages |
| Backend Hosting | Supabase |

**Node.js + Express: Removed.**

**Supabase Edge Functions: Primary Backend.**

**Supabase Realtime: Primary Live Synchronization.**

### Definition of Done

Aplikasi dapat menerima pesan teks pelanggan melalui Meta WhatsApp Cloud API, menyimpan kontak dan pesan ke Supabase, menampilkan percakapan secara realtime, serta mengirim balasan melalui Supabase Edge Functions tanpa backend Node.js terpisah.

### Important Public Access Note

Public database policies dan endpoint pengiriman pesan tanpa autentikasi berarti data serta kemampuan pengiriman dapat diakses pihak lain. Ini merupakan konsekuensi dari konfigurasi public yang dipilih. Meta credentials tetap server-side dan webhook tetap harus memverifikasi signature Meta. Tidak ada Supabase Auth yang ditambahkan.
