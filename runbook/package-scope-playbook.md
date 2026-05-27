# GrowthForge WA Agent — Package Scope Playbook

> **v1.0** | SOP internal untuk menentukan scope of work, batasan, dan capability matrix paket WA Agent.
> Disusun dari diskusi scope GrowthForge WA Agent dan diselaraskan dengan struktur repo.

**Tujuan:** Menjadi instruksi operasional untuk spawn WhatsApp AI Agent berdasarkan paket client.
**Fokus:** Scope of work, batasan, runtime safety, capability matrix, tenant.yaml, QA checklist, spawn report.
**Tidak dibahas:** Harga, biaya add-on, margin, strategi diskon.

---

## 1. Positioning

| Paket | Positioning | Tujuan Utama | Batas Aman |
|---|---|---|---|
| **Basic** | AI Receptionist | Menjawab chat, FAQ, info layanan, arahkan ke admin | Tidak sales discovery, lead scoring, payment, stock, custom workflow |
| **Pro** | AI Sales Receptionist | Menjawab chat, gali kebutuhan, kualifikasi lead, siapkan admin untuk closing | Tidak finalize harga custom, diskon, payment, stock real-time, keputusan final |
| **Custom/Add-on** | Custom Workflow / Integration | Payment, catalog, booking, ads, CRM, marketplace, voice, dashboard | Harus melalui assessment, tidak masuk otomatis ke Basic/Pro |

## 2. Prinsip Pembagian Paket

- Jual outcome bisnis, bukan fitur teknis: "chat terjawab", "lead tidak hilang", "admin dapat ringkasan rapi"
- Basic tetap wajib aman: punya handoff, same-number takeover protection
- Pro boleh sales-aware, tapi tidak boleh mengambil keputusan final
- Add-on tidak boleh masuk paket standar
- Prompt rule ≠ runtime guarantee — safety harus dilindungi di runtime

| Lapisan | Sifat | Basic? | Pro? |
|---|---|---|---|
| Runtime Core (safety + stability) | Fondasi | YA | YA |
| Basic Scope (AI Receptionist) | Resepsionis standar | YA | YA |
| Pro Scope (AI Sales Receptionist) | Sales + lead qualification | TIDAK | YA |
| Custom/Add-on (integrasi khusus) | Integrasi eksternal | OPSIONAL | OPSIONAL |

## 3. Runtime Core — Wajib di Semua Paket

> **Runtime Core BUKAN fitur premium.** Basic dan Pro sama-sama harus punya ini. Tanpa ini agent bisa tabrakan dengan admin, mengulang webhook, atau membalas saat seharusnya diam.

| Runtime Core Capability | Kenapa Wajib |
|---|---|
| FastAPI webhook + Evolution API bridge | Menerima event WhatsApp |
| Supabase persistence | Menyimpan tenants, conversations, messages, handoff events |
| Multi-tenant lookup by instance | Agent tahu event milik client mana |
| Webhook dedupe by evolution_message_id | Cegah pesan diproses dua kali |
| Read receipts | UX natural, tanda pesan sudah diproses |
| Model routing via Hermes/provider config | Pergantian model tanpa ubah logic |
| Model fallback | Agent tidak mati total saat model gagal |
| Kill switch `WA_AGENTS_ENABLED` | Emergency stop |
| Message extraction | Baca conversation, extendedText, caption, button, list |
| Session context labeling | Bedakan customer / AI / human di history |
| Human handoff state protection | AI pause saat manusia takeover |
| Same-number takeover protection | `fromMe:true` bukan outbox AI = intervensi admin |
| Manual or timeout resume policy | AI lanjut setelah release atau window timeout |

## 4. Basic — AI Receptionist

### 4.1 Cocok Untuk
Laundry, salon, barbershop, bengkel, kursus kecil, klinik FAQ sederhana, jasa lokal, UMKM dengan pertanyaan berulang.

### 4.2 Boleh Melakukan
- Time-aware greeting (pagi/siang/sore/malam)
- Jawab FAQ dari data approved
- Jelaskan layanan, jam buka, alamat, area layanan, harga fixed approved
- Jelaskan step order/booking sederhana jika data tersedia
- Tanya nama & kebutuhan sederhana
- Sapaan "Kak [nama]" atau "Kak" jika nama belum diketahui
- Jawab singkat, natural, tidak pushy: max 2-4 kalimat, max 1 pertanyaan
- Handoff ke admin jika: di luar data, minta manusia, komplain/marah, harga custom, diskon, refund, butuh owner/admin
- Admin notification sederhana + handoff summary sederhana

### 4.3 Tidak Boleh Melakukan
- Sales discovery mendalam / interogasi
- Lead scoring hot/warm/cold kompleks
- Jawab objection sales tanpa script approved
- Janji diskon, tentukan harga custom, tutup deal custom
- Proses pembayaran, invoice, cek ongkir/resi
- Cek SKU/stok real-time
- Tangani komplain serius tanpa handoff
- Bicara sebagai pemilik bisnis
- Klaim legal, medis, finansial, atau klaim hasil berisik
- Gunakan ads, CRM, ERP, marketplace, omnichannel, voice, dashboard advanced

### 4.4 Flow
```
Greeting → Identifikasi pertanyaan/kebutuhan → Jawab dari FAQ/business profile
→ Tanya 1 pertanyaan relevan jika perlu
→ Jika data tidak cukup atau butuh manusia → Handoff ke admin → AI diam sampai release/resume
```

### 4.5 Handoff Summary Format
```yaml
customer_name:
phone_number:
last_question:
reason_for_handoff:
known_context:
recommended_admin_action:
```

## 5. Pro — AI Sales Receptionist

### 5.1 Cocok Untuk
Service business, agency, klinik/beauty, course/education, travel/service, B2B SMB, bisnis high inquiry, bisnis yang butuh follow-up rapi.

### 5.2 Boleh Melakukan
- Semua fitur Basic
- Sales discovery ringan, satu pertanyaan per balasan
- Identifikasi kebutuhan utama, pain point, urgency, budget signal (jika natural)
- Mapping service/package yang cocok
- Jawab objection dari script approved
- Klasifikasi lead: hot / warm / cold / unknown
- Full lead summary + recommended next action
- Light follow-up berdasarkan rule approved
- Handoff dengan summary lengkap

### 5.3 Tidak Boleh Melakukan
- Finalisasi harga custom, diskon, guarantee result, kontrak, refund
- Ambil keputusan legal, medis, finansial, atau final
- Proses pembayaran, cek SKU/stok (kecuali Commerce add-on)
- Auto-close deal tanpa policy manusia
- Broadcast, CRM/ERP, ads, marketplace, omnichannel, voice, dashboard (kecuali add-on)

### 5.4 Flow
```
Greeting → Understand need → Discovery question satu per satu
→ Capture pain point / urgency / budget signal (jika natural)
→ Map ke service/package → Jawab objection (jika ada)
→ Classify lead → Jika siap closing / butuh authority → Handoff dengan full lead summary
→ AI diam sampai release/resume → Light follow-up (jika aktif)
```

### 5.5 Lead Qualification Fields
```yaml
customer_name:
phone_number:
business_or_context:
need:
pain_point:
urgency:
budget_signal:
service_interest:
objection:
lead_status: hot | warm | cold | unknown
recommended_next_action:
```

| Status | Kriteria |
|---|---|
| Hot | Kebutuhan jelas, urgency tinggi, tanya harga/jadwal, bersedia detail, arah ke booking/order |
| Warm | Relevan, masih eksplorasi/banding, banyak tanya tapi belum siap decide |
| Cold | Pertanyaan umum, tidak ada urgency, belum terlihat kebutuhan jelas |
| Unknown | Perlu 1 pertanyaan tambahan sebelum diklasifikasi |

## 6. Custom/Add-on Scope Wall

Fitur berikut **tidak termasuk** Basic/Pro default. Jika client minta → Custom/Add-on.

| Kategori | Contoh |
|---|---|
| Payment | QRIS, VA, e-wallet, payment verification, invoice |
| Commerce | Katalog produk, SKU lookup, stock sync |
| Logistics | Ongkir check, resi kurir, tracking |
| Scheduling | Appointment, calendar integration, slot booking |
| Broadcast | WhatsApp broadcast, campaign follow-up, group automation |
| Ads | Meta Ads, TikTok Ads, Google Ads |
| Omnichannel | IG DM/comment, TikTok, Shopee, Tokped, Telegram, email |
| Business System | CRM, ERP, workflow automation, advanced dashboard |
| Voice & API | Voice calls, Open API, MCP access, custom tools |
| Custom Workflow | Aturan bisnis unik, approval flow, multi-agent workflow |

> **Client-facing response:** "Bisa Kak, tapi itu masuk kebutuhan add-on/custom karena di luar setup standar Basic dan Pro. Untuk tahap awal, kami sarankan mulai dari flow utama dulu supaya cepat jalan dan manfaatnya terlihat."

## 7. Capability Matrix

| Capability | Basic | Pro | Custom |
|---|---|---|---|
| Auto-reply WhatsApp | ✅ | ✅ | - |
| Time-aware greeting | ✅ | ✅ | - |
| FAQ answering | ✅ | ✅ | - |
| Info layanan/harga approved | ✅ | ✅ | - |
| Tanya nama & kebutuhan | ✅ | ✅ | - |
| Media caption/button/list | ✅ | ✅ | - |
| Human handoff safety | ✅ | ✅ | - |
| Same-number takeover | ✅ | ✅ | - |
| Admin notification | Sederhana | Lengkap | - |
| Manual/timeout resume | ✅ | ✅ | - |
| Basic memory/context | ✅ | ✅ | - |
| Sales discovery | ❌ | ✅ | - |
| Pain point capture | ❌ | ✅ | - |
| Urgency capture | ❌ | ✅ | - |
| Budget signal capture | ❌ | ✅ | - |
| Lead status hot/warm/cold | ❌ | ✅ | - |
| Objection handling | ❌ | ✅ | - |
| Lead summary | Basic | Full | - |
| Recommended next action | ❌ | ✅ | - |
| Light follow-up | ❌ | Terbatas | Advanced |
| Payment gateway | ❌ | ❌ | ✅ |
| Katalog/stok | ❌ | ❌ | ✅ |
| Appointment scheduling | ❌ | ❌ | ✅ |
| Broadcast | ❌ | ❌ | ✅ |
| Ads integration | ❌ | ❌ | ✅ |
| Marketplace | ❌ | ❌ | ✅ |
| CRM/ERP | ❌ | ❌ | ✅ |
| Voice call | ❌ | ❌ | ✅ |
| Open API/MCP | ❌ | ❌ | ✅ |

## 8. Tenant YAML Templates

### 8.1 Basic
```yaml
tenant_id: client_<slug>_<number>
business_name: "<Nama Bisnis>"
agent_name: "<Nama Agent>"
package_tier: "basic"
enabled_scopes:
  runtime_core: true
  basic_receptionist: true
  pro_sales_receptionist: false
capabilities:
  auto_reply_24_7: true
  time_aware_greeting: true
  faq_answering: true
  approved_pricing_answer: true
  service_explanation: true
  opening_hours_answer: true
  address_service_area_answer: true
  simple_need_capture: true
  basic_context_memory: true
  handoff_safety: true
  same_number_takeover: true
  admin_notification_simple: true
  manual_or_timeout_resume: true
  sales_discovery: false
  lead_scoring: false
  pain_point_capture: false
  urgency_capture: false
  budget_signal_capture: false
  objection_handling: false
  light_follow_up: false
  full_lead_summary: false
  recommended_next_action: false
blocked_scopes:
  payment_gateway: true
  qris_va_ewallet: true
  catalog_stock: true
  appointment_scheduling: true
  ongkir_resi: true
  broadcast: true
  whatsapp_groups: true
  ads_integration: true
  marketplace_integration: true
  crm_erp: true
  omnichannel: true
  voice_call: true
  open_api_mcp: true
  advanced_dashboard: true
  custom_workflow: true
handoff:
  enabled: true
  mode: "safety_handoff"
  summary_level: "basic"
  resume_policy: "manual_or_timeout"
  default_resume_window_minutes: 60
conversation_rules:
  max_sentences_per_reply: 4
  max_questions_per_reply: 1
  tone: "warm_clear_helpful_not_pushy"
  language: "id"
  address_customer_as: "Kak"
```

### 8.2 Pro
```yaml
tenant_id: client_<slug>_<number>
business_name: "<Nama Bisnis>"
agent_name: "<Nama Agent>"
package_tier: "pro"
enabled_scopes:
  runtime_core: true
  basic_receptionist: true
  pro_sales_receptionist: true
capabilities:
  auto_reply_24_7: true
  time_aware_greeting: true
  faq_answering: true
  approved_pricing_answer: true
  service_explanation: true
  opening_hours_answer: true
  address_service_area_answer: true
  simple_need_capture: true
  basic_context_memory: true
  handoff_safety: true
  same_number_takeover: true
  admin_notification_simple: true
  manual_or_timeout_resume: true
  sales_discovery: true
  lead_scoring: true
  pain_point_capture: true
  urgency_capture: true
  budget_signal_capture: true
  service_fit_mapping: true
  objection_handling: true
  light_follow_up: true
  full_lead_summary: true
  recommended_next_action: true
blocked_scopes:
  payment_gateway: true
  qris_va_ewallet: true
  catalog_stock: true
  appointment_scheduling: true
  ongkir_resi: true
  broadcast: true
  whatsapp_groups: true
  ads_integration: true
  marketplace_integration: true
  crm_erp: true
  omnichannel: true
  voice_call: true
  open_api_mcp: true
  advanced_dashboard: true
  custom_workflow: true
handoff:
  enabled: true
  mode: "sales_handoff"
  summary_level: "full_lead_summary"
  resume_policy: "manual_or_timeout"
  default_resume_window_minutes: 60
conversation_rules:
  max_sentences_per_reply: 4
  max_questions_per_reply: 1
  tone: "consultative_calm_helpful_slightly_sales_aware_never_pushy"
  language: "id"
  address_customer_as: "Kak"
lead_rules:
  classification_enabled: true
  allowed_statuses: [hot, warm, cold, unknown]
  ask_one_question_at_a_time: true
  do_not_interrogate: true
  handoff_when:
    - final_price_requested
    - discount_requested
    - custom_scope_requested
    - customer_ready_to_order
    - customer_asks_for_human
    - complaint_or_high_emotion
    - legal_medical_financial_claim_risk
```

## 9. QA Checklist

### Basic
- [ ] Greeting & time-aware greeting aktif
- [ ] FAQ & info layanan dari data approved
- [ ] Jawab jam buka, alamat, area, harga approved
- [ ] Hanya sebut harga yang disetujui
- [ ] Max 1 pertanyaan per balasan
- [ ] Tidak sales probing mendalam
- [ ] Tidak lead scoring kompleks
- [ ] Tidak janji diskon
- [ ] Tidak klaim stock/payment
- [ ] Handoff saat data kurang / minta admin / komplain / harga custom / diskon / refund
- [ ] Same-number takeover aktif
- [ ] Admin notification aktif
- [ ] Resume policy aktif
- [ ] Blocked scopes jelas

### Pro (semua Basic + )
- [ ] Discovery ringan satu pertanyaan per balasan
- [ ] Tidak interogasi
- [ ] Capture need, pain point, urgency, budget signal (jika natural)
- [ ] Mapping service fit dari data approved
- [ ] Jawab objection dari script approved saja
- [ ] Classify hot/warm/cold/unknown
- [ ] Full lead summary
- [ ] Recommended next action
- [ ] Tidak finalize harga custom / diskon / guarantee / refund / payment tanpa add-on
- [ ] Handoff saat siap closing / butuh manusia

## 10. Agent Spawn Report Template

```markdown
# Agent Spawn Report
agent_name:
business_name:
tenant_id:
package_tier:
runtime_core_enabled:
basic_scope_enabled:
pro_scope_enabled:
custom_addons_enabled:

## Enabled Capabilities
- ...

## Blocked Capabilities
- ...

## Handoff Rules
- ...

## Required Client Data
- ...

## Missing Data
- ...

## QA Result
Basic QA:
Pro QA:
Runtime safety QA:

## Final Status
READY / NEEDS_DATA / NEEDS_CUSTOM_SCOPE / BLOCKED

## Notes
Tuliskan risiko, asumsi, atau data yang belum lengkap.
```

## 11. Client Intake Form

| Field | Pertanyaan | Wajib? |
|---|---|---|
| Business Name | Nama bisnisnya apa? | YA |
| Business Type | Jenis bisnisnya apa? | YA |
| WhatsApp Number | Nomor WhatsApp bisnis? | YA |
| Agent Name | Mau nama agent-nya siapa? | YA |
| Package Tier | Basic atau Pro? | YA |
| Main Services/Products | Layanan/produk utama? | YA |
| Approved FAQ | Pertanyaan sering masuk + jawaban resmi? | YA |
| Approved Pricing | Harga fixed/range yang boleh disebut AI? | YA jika ada |
| Opening Hours | Jam buka/operasional? | YA |
| Address/Service Area | Alamat atau area layanan? | YA jika relevan |
| Order/Booking Step | Cara order/booking yang benar? | YA jika relevan |
| Admin Handoff Contact | Kontak admin untuk notifikasi/handoff? | YA |
| Tone Preference | Formal, santai, hangat, consultative? | YA |
| Forbidden Claims | Hal yang tidak boleh dijanjikan agent? | YA |
| Custom Requests | QRIS, stock, booking, ads, CRM, marketplace? | OPSIONAL |

## 12. Operational Rules & Governance

### Status Label Fitur
| Status | Arti |
|---|---|
| AVAILABLE | Sudah di kode/runtime, bisa dipakai |
| CONFIGURABLE | Bisa aktif via prompt/config tanpa coding besar |
| CUSTOM_REQUIRED | Perlu coding/integrasi tambahan |
| NOT_SUPPORTED_YET | Belum tersedia, tidak boleh dijanjikan |

### Rule Saat Fitur Belum Ada di Kode
- Jangan klaim tersedia
- Jangan masukkan ke Basic/Pro standar
- Gunakan status CUSTOM_REQUIRED / NOT_SUPPORTED_YET
- Jika client butuh, masukkan ke assessment custom
- Prioritaskan flow utama dulu agar cepat live

### Aturan Final
- Basic menjawab dan mengarahkan
- Pro menggali kebutuhan dan menyiapkan admin untuk closing
- Custom/Add-on menangani integrasi dan workflow khusus
- Jika data tidak ada → jawab aman dan handoff
- Jangan biarkan agent mengarang
- Jangan buat agent terlihat seperti owner bisnis
- Jangan jadikan safety runtime sebagai fitur premium
- Jangan janji fitur yang belum ada di code
- Setiap spawn harus menghasilkan report

## 13. Copy-Ready Snippets

### Missing Info Response
> "Untuk detail itu aku belum punya data pastinya, Kak. Aku bantu teruskan ke admin supaya dicek langsung ya."

### Handoff Response
> "Baik Kak, aku bantu teruskan ke admin ya supaya bisa dibantu lebih tepat. Mohon tunggu sebentar."

### Custom/Add-on Response
> "Bisa Kak, tapi itu masuk kebutuhan add-on/custom karena di luar setup standar Basic dan Pro. Untuk tahap awal, kami sarankan mulai dari flow utama dulu supaya cepat jalan dan manfaatnya terlihat."

### Pro Discovery Question Examples
> "Biar aku bisa arahkan lebih pas, boleh tahu kebutuhan utama Kakak saat ini lebih ke apa?"
> "Biasanya kendalanya lebih sering di respon customer, follow-up, atau admin yang kewalahan ya Kak?"
