WhatsApp Cloud API Integration

Technical Specification — V1.6

Repository: gantulu/whatsdillah
Version: 1.6
Status: Draft — Revisi dari V1.4
Architecture: React + Vite + TypeScript + Supabase Edge Functions + PostgreSQL + Realtime
WhatsApp Integration: Official Meta WhatsApp Cloud API

1. Project Overview

Membangun aplikasi WhatsApp Management berbasis WhatsApp Cloud API resmi Meta.

Aplikasi digunakan untuk menerima pesan pelanggan, menampilkan percakapan, mengirim balasan, menyimpan kontak dan riwayat pesan, serta memperbarui status pesan secara realtime.

1.1 Main Features

Menerima pesan teks pelanggan melalui Meta Webhook.

Menampilkan daftar percakapan.

Menampilkan riwayat percakapan.

Mengirim balasan pesan teks kepada pelanggan.

Menyimpan kontak dan riwayat pesan.

Memperbarui status pesan secara realtime.

Menampilkan informasi kontak dan waktu pesan terakhir.

1.2 Architecture Principles

Tidak menggunakan Node.js + Express.

Backend menggunakan Supabase Edge Functions.

Runtime Edge Functions menggunakan Deno.

Database menggunakan Supabase PostgreSQL.

Sinkronisasi realtime menggunakan Supabase Realtime.

Frontend menggunakan Supabase JavaScript Client.

Tidak menggunakan Supabase Auth.

Tidak menggunakan login pengguna.

Akses aplikasi dan database bersifat public.

Tidak menambahkan sistem keamanan tambahan.

Supabase URL dan Publishable Key ditempatkan di frontend.

Meta credentials tetap berada di Supabase Edge Function Secrets.

Integrasi WhatsApp menggunakan API resmi Meta.

2. Fixed Architecture Decisions

Keputusan berikut menjadi dasar implementasi V1.6.

Component

Decision

Frontend

React

Build Tool

Vite

Language

TypeScript

UI

WhatsApp Mobile Design System

Database

Supabase PostgreSQL

Database Client

Supabase JS Client

Backend

Supabase Edge Functions

Runtime

Deno

Realtime

Supabase Realtime

WhatsApp API

Meta WhatsApp Cloud API

Authentication

None

Login

Not Required

Database Access

Public

RLS

Public policies

Additional Security System

None

Frontend Hosting

GitHub Pages

Backend Hosting

Supabase

2.1 Public Access

Definisi public untuk aplikasi ini:
- Siapa saja dapat membuka dan menggunakan aplikasi melalui URL publik.
- Tidak ada login, registrasi, akun pengguna, atau Supabase Auth.
- Tidak ada pembatasan penggunaan aplikasi berdasarkan identitas pengguna.
- Semua pengunjung menggunakan akses database melalui role anon.
- Operasi baca/tulis database mengikuti public grants dan RLS policies yang ditetapkan pada spesifikasi ini.
- Endpoint whatsapp-send-message dapat dipanggil tanpa user login.
- Tidak ada pemisahan data atau kepemilikan berdasarkan pengguna.

Aplikasi ini memang dirancang sebagai aplikasi WhatsApp Management yang terbuka untuk umum. Public bukan sekadar berarti halaman frontend dapat dilihat; pengunjung juga dapat menggunakan fitur aplikasi sesuai operasi yang disediakan.


Konfigurasi akses:

Frontend dapat menggunakan Supabase Publishable Key.

Database menggunakan public access.

Role anon digunakan untuk akses database tanpa login.

Public RLS policies diterapkan pada tabel contacts dan messages.

Edge Functions dapat dipanggil melalui endpoint publik.

Tidak ada Supabase Auth atau user session requirement.

Konsekuensi akses public: pihak lain yang mengetahui Supabase Project URL dan Publishable Key dapat melakukan operasi yang diizinkan oleh grants dan RLS policies.

3. Technology Stack

Layer

Technology

Frontend

React

Build Tool

Vite

Language

TypeScript

UI Design

WhatsApp Mobile Design System

Database

Supabase PostgreSQL

Database Client

@supabase/supabase-js

Backend

Supabase Edge Functions

Runtime

Deno

Realtime

Supabase Realtime

WhatsApp API

Meta WhatsApp Cloud API

Frontend Hosting

GitHub Pages

Backend Hosting

Supabase

Source Control

GitHub

3.1 Frontend Dependencies

npm install @supabase/supabase-js

React, Vite, dan TypeScript digunakan sebagai fondasi aplikasi frontend.

4. System Architecture

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

4.1 Component Responsibilities

Component

Responsibility

React

Antarmuka aplikasi

Supabase Client

Akses database dan pemanggilan Edge Functions

Edge Functions

Backend dan integrasi Meta

PostgreSQL

Penyimpanan kontak dan pesan

Realtime

Sinkronisasi perubahan database

Meta Cloud API

Pengiriman dan penerimaan pesan

4.2 Data Flow

Incoming message:

Meta Cloud API → whatsapp-webhook → PostgreSQL → Supabase Realtime → React Frontend.

Outgoing message:

React Frontend → whatsapp-send-message → Meta Graph API → PostgreSQL → Supabase Realtime → React Frontend.

Message status:

Meta Cloud API → whatsapp-webhook → PostgreSQL → Supabase Realtime → React Frontend.

5. Supabase Client

5.1 Configuration

File:

frontend/src/lib/supabase.ts

import { createClient } from '@supabase/supabase-js'

const supabaseUrl =
  'https://YOUR_PROJECT_ID.supabase.co'

const supabasePublishableKey =
  'sb_publishable_YOUR_KEY'

export const supabase = createClient(
  supabaseUrl,
  supabasePublishableKey
)

5.2 Requirements

Supabase URL ditentukan di frontend.

Publishable Key ditentukan di frontend.

Tidak menggunakan Supabase Auth.

Tidak memerlukan login.

Tidak memasukkan Meta Access Token ke frontend.

Tidak memasukkan Supabase Secret Key ke frontend.

6. Database Design

V1.6 tetap menggunakan dua tabel utama:

contacts

messages

Tidak ada penambahan tabel bisnis baru dalam cakupan V1.5.

6.1 Contacts Table

Column

Type

Description

id

UUID

Primary Key

phone_number

TEXT

Nomor WhatsApp

name

TEXT

Nama kontak

last_message_at

TIMESTAMPTZ

Waktu pesan terakhir

created_at

TIMESTAMPTZ

Waktu dibuat

updated_at

TIMESTAMPTZ

Waktu diperbarui

Constraints:

id merupakan Primary Key.

phone_number wajib diisi.

phone_number bersifat unique.

created_at memiliki default NOW().

updated_at memiliki default NOW().

6.2 Messages Table

Column

Type

Description

id

UUID

Primary Key

contact_id

UUID

Foreign Key ke contacts

wamid

TEXT

Meta Message ID

direction

TEXT

incoming / outgoing

message_type

TEXT

text

message

TEXT

Isi pesan

status

TEXT

Status pengiriman

created_at

TIMESTAMPTZ

Waktu dibuat

updated_at

TIMESTAMPTZ

Waktu diperbarui

Constraints:

id merupakan Primary Key.

contact_id wajib diisi.

contact_id mengacu ke contacts.id.

wamid unique apabila tersedia.

direction hanya menerima incoming atau outgoing.

message_type untuk V1.6 hanya menerima text.

message wajib diisi.

status menggunakan status pesan yang ditentukan pada bagian Message Status.

6.3 Database SQL

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
    status TEXT DEFAULT 'pending'
        CHECK (
            status IN (
                'pending',
                'sent',
                'delivered',
                'read',
                'failed'
            )
        ),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_messages_contact_id
ON public.messages(contact_id);

CREATE INDEX idx_messages_created_at
ON public.messages(created_at DESC);

6.4 Updated Timestamp

Kolom updated_at diperbarui ketika record mengalami perubahan.

CREATE OR REPLACE FUNCTION public.update_updated_at()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$;

CREATE TRIGGER contacts_updated_at
BEFORE UPDATE ON public.contacts
FOR EACH ROW
EXECUTE FUNCTION public.update_updated_at();

CREATE TRIGGER messages_updated_at
BEFORE UPDATE ON public.messages
FOR EACH ROW
EXECUTE FUNCTION public.update_updated_at();

6.5 Last Message Timestamp

contacts.last_message_at diperbarui ketika pesan baru berhasil disimpan.

Nilainya mengikuti waktu pesan yang paling baru untuk kontak tersebut.

7. Public Database Access

7.1 Configuration

Setting

Value

Supabase Auth

Not Used

Login

Not Required

Database Access

Public

Database Role

anon

Publishable Key

Frontend

RLS

Enabled

Public Policies

Enabled

Additional User Authentication

None

Additional Security System

None

7.2 Public RLS Policies

ALTER TABLE public.contacts
ENABLE ROW LEVEL SECURITY;

ALTER TABLE public.messages
ENABLE ROW LEVEL SECURITY;

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

7.3 Public Access Behavior

Role anon dapat mengakses data sesuai grants dan policies.

Operasi database tidak memerlukan login.

Frontend menggunakan Publishable Key.

Tidak ada pemisahan data berdasarkan user.

Kontak dan pesan berada dalam database yang sama.

Semua akses mengikuti public access configuration.

7.4 Public Access Consequence

Data yang tersedia melalui public database access dapat dibaca atau diubah pihak lain sesuai izin database yang berlaku.

Kemampuan mengirim pesan melalui endpoint publik juga dapat digunakan pihak lain.

Kondisi tersebut merupakan konsekuensi dari arsitektur public yang dipilih.

8. Supabase Edge Functions

Supabase Edge Functions menjadi backend utama aplikasi.

Semua function menggunakan TypeScript dengan Deno runtime.

8.1 Function Structure

supabase/
└── functions/
    ├── whatsapp-webhook/
    │   └── index.ts
    │
    └── whatsapp-send-message/
        └── index.ts

8.2 Function Responsibilities

Function

Responsibility

whatsapp-webhook

Menerima pesan masuk dan status pesan dari Meta

whatsapp-send-message

Mengirim pesan kepada pelanggan melalui Meta Graph API

8.3 Webhook Endpoint

GET  /functions/v1/whatsapp-webhook
POST /functions/v1/whatsapp-webhook

Satu callback URL menangani kedua metode HTTP:

GET

Menerima webhook verification request.

Memeriksa hub.mode.

Memeriksa hub.verify_token.

Mengembalikan hub.challenge jika verifikasi berhasil.

POST

Menerima event dari Meta.

Memproses pesan masuk.

Memproses perubahan status pesan.

Menyimpan perubahan ke PostgreSQL.

Tanggung jawab whatsapp-webhook-verify pada V1.4 dikonsolidasikan ke dalam GET handler pada whatsapp-webhook, karena Meta menggunakan satu callback URL untuk proses verifikasi dan pengiriman event.

8.4 Send Message Endpoint

POST /functions/v1/whatsapp-send-message

Endpoint ini dapat dipanggil langsung dari frontend melalui Supabase Client.

Request:

{
  "contact_id": "contact-001",
  "message": "Halo! Produk masih tersedia."
}

Response sukses:

{
  "success": true,
  "data": {
    "message_id": "wamid.example",
    "status": "sent"
  }
}

Response gagal:

{
  "success": false,
  "error": {
    "code": "MESSAGE_SEND_FAILED",
    "message": "Message could not be sent."
  }
}

8.5 Function Configuration

Karena aplikasi tidak menggunakan Supabase Auth, function yang dipanggil tanpa user JWT menggunakan konfigurasi tanpa pemeriksaan JWT.

Contoh supabase/config.toml:

[functions.whatsapp-webhook]
verify_jwt = false

[functions.whatsapp-send-message]
verify_jwt = false

Tidak ada kewajiban mengirim Supabase user JWT untuk menggunakan kedua endpoint tersebut.

9. Receive Messages

9.1 Incoming Message Flow

Customer sends message
          |
          v
Meta WhatsApp Cloud API
          |
          v
whatsapp-webhook
          |
          v
Receive webhook payload
          |
          v
Process incoming message
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

9.2 Requirements

Menerima pesan melalui Meta Webhook.

Mendukung pesan teks.

Membuat kontak otomatis jika belum tersedia.

Menggunakan nomor WhatsApp sebagai identitas kontak.

Menyimpan pesan ke PostgreSQL.

Menyimpan WAMID dari Meta.

Mencegah penyimpanan pesan yang sama berulang kali berdasarkan WAMID.

Memperbarui last_message_at.

Menampilkan pesan baru melalui Realtime.

9.3 Incoming Message Data

Pesan masuk disimpan dengan:

Field

Value

contact_id

ID kontak

wamid

Meta Message ID

direction

incoming

message_type

text

message

Isi pesan

status

Status internal yang sesuai

10. Reply Messages

10.1 Meta Graph API

POST https://graph.facebook.com/{VERSION}/{PHONE_NUMBER_ID}/messages

10.2 Outgoing Message Flow

Admin types message
         |
         v
React Frontend
         |
         v
whatsapp-send-message
         |
         v
Find contact
         |
         v
Create pending message
         |
         v
Meta Graph API
         |
         v
Meta API Response
         |
         v
Save WAMID and status
         |
         v
Supabase Realtime
         |
         v
Update Chat UI

10.3 Meta Request

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

10.4 Requirements

Pesan tidak boleh kosong.

Penerima menggunakan nomor internasional.

Nomor penerima berasal dari kontak yang dipilih.

Meta Access Token berada di Edge Function Secrets.

Simpan WAMID dari respons Meta.

Simpan status pengiriman.

Tampilkan error jika Meta API gagal.

Patuhi customer service window dan ketentuan template Meta.

Pesan keluar disimpan ke PostgreSQL.

Pesan keluar ditampilkan melalui Realtime.

10.5 Send Message Result

Jika Meta menerima permintaan pengiriman:

Simpan WAMID.

Perbarui status menjadi sent.

Sinkronkan perubahan ke frontend.

Jika Meta mengembalikan kegagalan:

Perbarui status menjadi failed.

Simpan informasi error yang diperlukan.

Tampilkan kegagalan pada UI.

11. Supabase Realtime

Supabase Realtime digunakan untuk menyinkronkan perubahan database dengan frontend melalui WebSocket.

11.1 Realtime Tables

Tabel yang digunakan:

public.contacts

public.messages

contacts digunakan untuk memperbarui daftar percakapan.

messages digunakan untuk memperbarui isi percakapan dan status pesan.

11.2 Enable Realtime

ALTER PUBLICATION supabase_realtime
ADD TABLE public.contacts;

ALTER PUBLICATION supabase_realtime
ADD TABLE public.messages;

11.3 Frontend Subscription

import { supabase } from '@/lib/supabase'

const channel = supabase
  .channel('whatsapp-realtime')
  .on(
    'postgres_changes',
    {
      event: '*',
      schema: 'public',
      table: 'contacts'
    },
    (payload) => {
      console.log('Contact update:', payload)
    }
  )
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

11.4 Realtime Events

Event

Usage

INSERT

Kontak atau pesan baru

UPDATE

Perubahan data kontak atau status pesan

DELETE

Penghapusan data

11.5 Realtime Requirements

Pesan masuk langsung muncul.

Pesan keluar langsung muncul.

Kontak baru langsung muncul pada daftar percakapan.

Perubahan nama kontak dapat disinkronkan.

Status sent diperbarui.

Status delivered diperbarui.

Status read diperbarui.

Status failed diperbarui.

UI tidak memerlukan refresh manual.

Hindari duplikasi data pada frontend.

Unsubscribe ketika komponen dilepas.

Realtime hanya menyinkronkan perubahan database. Pengiriman pesan tetap dilakukan melalui Edge Function dan Meta Graph API.

12. Message Status

Status pesan yang digunakan:

Status

Description

pending

Menunggu proses pengiriman

sent

Pesan diterima Meta untuk pengiriman

delivered

Pesan telah sampai ke perangkat penerima

read

Pesan telah dibaca penerima

failed

Pengiriman pesan gagal

12.1 Status Update Flow

Meta WhatsApp Cloud API
          |
          v
Status Webhook
          |
          v
whatsapp-webhook
          |
          v
Find message by WAMID
          |
          v
Update message status
          |
          v
Supabase Realtime
          |
          v
React Chat UI

12.2 Status Requirements

Status pesan mengikuti event status dari Meta.

WAMID digunakan untuk mengidentifikasi pesan.

Status yang diterima diperbarui pada record pesan.

Perubahan status dikirim melalui Realtime.

Status pesan tidak boleh menyebabkan record pesan duplikat.

12.3 Webhook and Message Ordering

Pemrosesan event memperhatikan kemungkinan urutan event Meta berbeda dari urutan penyimpanan pesan lokal.

Jika event status diterima sebelum record pesan ditemukan:

Identifikasi pesan berdasarkan WAMID.

Lakukan pencocokan kembali dengan data pesan.

Pastikan status akhir tetap diperbarui.

Hindari membuat record pesan duplikat.

13. Webhook Requirements

13.1 Verification

GET request menerima parameter:

hub.mode

hub.verify_token

hub.challenge

Ketentuan:

Mode harus sesuai dengan permintaan subscription Meta.

Verify Token harus sama dengan konfigurasi META_VERIFY_TOKEN.

Jika valid, kembalikan challenge.

Jika tidak valid, kembalikan HTTP error.

13.2 Event Receiver

POST request menerima event dari Meta.

Event yang diproses:

Incoming text messages.

Message sent status.

Message delivered status.

Message read status.

Message failed status.

13.3 Signature Validation

Tetap mengikuti persyaratan integrasi webhook Meta pada V1.4:

Menggunakan header X-Hub-Signature-256.

Menggunakan META_APP_SECRET.

Memverifikasi signature terhadap raw request body.

Memproses payload setelah pemeriksaan signature berhasil.

Signature validation merupakan bagian dari verifikasi protokol webhook Meta yang sudah ditetapkan pada V1.4, bukan penambahan sistem autentikasi pengguna.

13.4 Duplicate Event Handling

WAMID digunakan sebagai identitas pesan.

Pesan yang sama tidak disimpan berulang kali.

Webhook retry tidak menghasilkan pesan duplikat.

Status pesan diperbarui berdasarkan event yang diterima.

13.5 Webhook Response

Webhook mengembalikan HTTP response yang sesuai dengan hasil penerimaan dan pemrosesan event.

Event yang sama dapat dikirim ulang oleh Meta, sehingga pemrosesan harus mempertahankan konsistensi data.

14. Environment Secrets

Credential integrasi Meta disimpan melalui Supabase Edge Function Secrets.

META_ACCESS_TOKEN=
META_APP_SECRET=
META_VERIFY_TOKEN=

WHATSAPP_PHONE_NUMBER_ID=
WHATSAPP_BUSINESS_ACCOUNT_ID=

META_GRAPH_API_VERSION=

Credential Supabase server-side yang diperlukan untuk operasi backend juga berada di lingkungan server.

Ketentuan:

Meta Access Token hanya digunakan di backend.

Meta App Secret hanya digunakan di backend.

Meta Verify Token hanya digunakan di backend.

WhatsApp Phone Number ID digunakan oleh Edge Function.

WhatsApp Business Account ID digunakan sesuai kebutuhan integrasi.

Tidak memasukkan credential rahasia ke frontend.

Tidak memasukkan credential rahasia ke repository GitHub.

15. Frontend UI

15.1 Main Screens

Screen

Features

Chat List

Daftar percakapan

Conversation

Riwayat pesan

Message Input

Menulis dan mengirim balasan

Contact Information

Informasi kontak

15.2 Chat List

Menampilkan:

Nama kontak.

Nomor WhatsApp.

Pesan terakhir.

Waktu pesan terakhir.

Indikator pesan belum dibaca.

15.3 Conversation

Menampilkan:

Incoming message bubbles.

Outgoing message bubbles.

Timestamp.

Message status.

Text input.

Send button.

15.4 UI Requirements

Desain mengikuti WhatsApp Mobile UI.

Tampilan dioptimalkan untuk mobile.

Daftar kontak dan percakapan diperbarui melalui Realtime.

Pesan masuk dan keluar menggunakan tampilan bubble berbeda.

Status pesan ditampilkan pada pesan keluar.

Pengiriman pesan menggunakan Edge Function.

16. Project Structure

whatsdillah/
│
├── frontend/
│   └── src/
│       ├── lib/
│       │   └── supabase.ts
│       │
│       ├── components/
│       │
│       ├── pages/
│       │
│       ├── features/
│       │   └── chat/
│       │       ├── components/
│       │       ├── hooks/
│       │       ├── services/
│       │       └── types/
│       │
│       ├── services/
│       └── types/
│
├── supabase/
│   ├── config.toml
│   │
│   ├── migrations/
│   │
│   └── functions/
│       ├── whatsapp-webhook/
│       │   └── index.ts
│       │
│       └── whatsapp-send-message/
│           └── index.ts
│
├── docs/
│   └── whatsapp-cloud-api/
│       ├── README.md
│       ├── v1.5-specification.md
│       ├── database.md
│       ├── webhook.md
│       ├── receive-messages.md
│       ├── reply-messages.md
│       ├── realtime.md
│       └── deployment.md
│
└── README.md

17. Deployment

17.1 Frontend Deployment

Platform: GitHub Pages.

Build menggunakan Vite.

npm install
npm run build

Hasil build berada di:

dist/

GitHub Pages menggunakan hasil build tersebut sebagai frontend aplikasi.

Vite base harus disesuaikan dengan nama repository dan URL GitHub Pages.

17.2 Backend Deployment

Platform: Supabase.

Deploy function:

supabase functions deploy whatsapp-webhook

supabase functions deploy whatsapp-send-message

17.3 Database Deployment

Database menggunakan Supabase PostgreSQL.

Schema dan perubahan database dikelola melalui Supabase migrations.

17.4 Realtime Deployment

Aktifkan Realtime pada:

public.contacts

public.messages

17.5 Meta Webhook Configuration

Callback URL:

https://YOUR_PROJECT_REF.supabase.co/functions/v1/whatsapp-webhook

Masukkan Verify Token yang sama dengan nilai META_VERIFY_TOKEN.

Webhook digunakan untuk menerima pesan masuk dan perubahan status pesan.

18. Testing Strategy

18.1 Database Testing

Tabel contacts berhasil dibuat.

Tabel messages berhasil dibuat.

Primary Key berfungsi.

Foreign Key berfungsi.

Unique phone number berfungsi.

Unique WAMID berfungsi.

Direction constraint berfungsi.

Message type constraint berfungsi.

Message status constraint berfungsi.

Updated timestamp diperbarui.

18.2 Public Access Testing

Tidak menggunakan Supabase Auth.

Tidak memerlukan login.

Publishable Key dapat digunakan dari frontend.

Public policies aktif.

Role anon dapat melakukan operasi sesuai konfigurasi public access.

18.3 Edge Functions Testing

whatsapp-webhook berjalan dengan Deno.

whatsapp-send-message berjalan dengan Deno.

Endpoint dapat dipanggil tanpa user login.

Webhook verification berhasil.

Webhook receiver berhasil.

Send message function berhasil.

Error response dapat ditampilkan pada frontend.

18.4 Receive Messages Testing

Pesan teks berhasil diterima.

Kontak otomatis dibuat.

Pesan tersimpan.

WAMID tersimpan.

Tidak ada duplikasi pesan.

last_message_at diperbarui.

18.5 Reply Messages Testing

Pesan dapat ditulis dari frontend.

Pesan dikirim melalui Meta Cloud API.

WAMID tersimpan.

Status awal diperbarui.

Error Meta API ditangani.

18.6 Webhook Testing

GET verification berhasil.

GET verification gagal jika Verify Token tidak cocok.

POST menerima event Meta.

Signature validation berjalan.

Incoming message berhasil diproses.

Status webhook berhasil diproses.

Retry event tidak menghasilkan pesan duplikat.

18.7 Realtime Testing

INSERT contacts berfungsi.

UPDATE contacts berfungsi.

INSERT messages berfungsi.

UPDATE messages berfungsi.

Pesan masuk langsung muncul.

Pesan keluar langsung muncul.

Status pesan langsung berubah.

UI tidak memerlukan refresh manual.

Subscription dilepas saat komponen tidak digunakan.

18.8 Frontend and Deployment Testing

React frontend berjalan.

Supabase Client terhubung.

Chat List berjalan.

Conversation berjalan.

Message Input berjalan.

GitHub Pages build berhasil.

Supabase Edge Functions berhasil di-deploy.

Meta Webhook berhasil terhubung.

19. Development Roadmap

Phase

Task

Deliverable

1

Supabase Project Setup

Supabase project siap

2

Database Schema

Contacts dan Messages

3

Public Access Configuration

Public RLS policies

4

Edge Functions Setup

Webhook dan send function

5

Meta Webhook Verification

GET verification

6

Receive Text Messages

Incoming messages

7

Supabase Realtime

Live synchronization

8

Send Text Messages

Outgoing messages

9

Message Status Synchronization

Status updates

10

React Frontend Integration

WhatsApp Management UI

11

Testing

Functional verification

12

Deployment

GitHub Pages dan Supabase

20. Official Documentation

Supabase

Supabase Edge Functions

Supabase Edge Functions Limits

Supabase Function Configuration

Supabase Realtime

Postgres Changes

Supabase API Keys

Supabase Row Level Security

Meta WhatsApp Cloud API

WhatsApp Business Platform

WhatsApp Cloud API

WhatsApp Cloud API Get Started

WhatsApp Webhooks

WhatsApp Send Messages

21. Final Architecture Decision

Component

Final Decision

Frontend

React + Vite + TypeScript

Backend

Supabase Edge Functions

Runtime

Deno

Database

Supabase PostgreSQL

Database Client

Supabase JS Client

Realtime

Supabase Realtime

WhatsApp API

Official Meta Cloud API

Authentication

None

Login

Not Required

Database Access

Public

Database Role

anon

RLS

Public policies

Additional Security System

None

Frontend Hosting

GitHub Pages

Backend Hosting

Supabase

Final Architecture Principles

Node.js + Express: Not Used.

Supabase Edge Functions: Primary Backend.

Supabase PostgreSQL: Primary Database.

Supabase Realtime: Primary Live Synchronization.

Meta WhatsApp Cloud API: Official WhatsApp Integration.

Authentication: None.

Database Access: Public.

Additional Security System: None.

22. Definition of Done

V1.6 dinyatakan selesai apabila aplikasi dapat:

Menampilkan aplikasi WhatsApp Management melalui React Frontend.

Mengakses Supabase PostgreSQL menggunakan public access.

Menerima pesan teks pelanggan melalui Meta WhatsApp Cloud API.

Membuat dan menyimpan kontak secara otomatis.

Menyimpan riwayat pesan.

Menampilkan daftar percakapan secara realtime.

Menampilkan pesan masuk dan pesan keluar.

Mengirim balasan melalui Supabase Edge Functions.

Menyimpan WAMID dan status pesan.

Memperbarui status pesan melalui webhook Meta.

Menampilkan perubahan database tanpa refresh manual.

Berjalan di GitHub Pages dengan backend di Supabase.

Aplikasi menggunakan akses public tanpa Supabase Auth, tanpa login, dan tanpa sistem keamanan tambahan sesuai keputusan arsitektur proyek.

END OF TECHNICAL SPECIFICATION V1.6
