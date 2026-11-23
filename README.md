1. Receive Messages
Official Meta
Incoming Messages

Meta – Webhooks DocumentationMeta – WhatsApp WebhooksFungsi:
- Menerima pesan masuk dari pengguna.
- Menerima gambar, video, audio, dan dokumen.
- Menerima informasi kontak dan lokasi.
- Menerima status pesan.
- Memproses event melalui HTTPS Webhook.
Webhook events:
- messages — pesan masuk.
- statuses — sent, delivered, read, failed.
2. Send Messages
Official Meta
Outgoing Messages

Meta – Send Messages DocumentationMeta – Messages API ReferenceEndpoint:
POST https://graph.facebook.com/{VERSION}/{PHONE_NUMBER_ID}/messages


Fungsi:
- Mengirim pesan teks.
- Mengirim gambar, video, audio, dokumen.
- Mengirim template messages.
- Mengirim interactive buttons.
- Mengirim interactive lists.
- Membalas pesan pelanggan.
Contoh request:
{
  "messaging_product": "whatsapp",
  "recipient_type": "individual",
  "to": "6281234567890",
  "type": "text",
  "text": {
    "body": "Hello from WhatsApp Cloud API!"
  }
}


3. Alur integrasi
Receive Messages

WhatsApp User → Meta Webhook → Backend → Database → Mobile UI

Send Messages

Mobile UI → Backend → Meta Graph API → WhatsApp User


4. Rekomendasi implementasi
Untuk repository WhatsApp, kedua dokumentasi ini menjadi acuan utama:
Modul	Implementasi
Receive	Webhook verification
Receive	Incoming message handler
Receive	Message persistence
Send	Text message service
Send	Media message service
Send	Template message service
Send	Interactive message service
Both	Message status synchronization

Catatan: Gunakan dokumentasi Meta sebagai sumber utama, bukan SDK Node.js lama yang sudah diarsipkan. 
whatsapp.github.io+2
