# 📋 Dokumentasi Logika Teknis — Portfolio Feyza Revalina

> **Dokumen ini menjabarkan arsitektur sistem, alur logika bisnis, dan strategi teknis dari lima proyek utama yang mencerminkan kemampuan Logical Thinking dalam merancang sistem kompleks dan terintegrasi.**

---

## Daftar Isi

1. [Sistem Inventaris — Madewi Cookies](#1-sistem-inventaris--madewi-cookies)
2. [Manajemen IKM — Kurnia Jati Furniture](#2-manajemen-ikm--kurnia-jati-furniture)
3. [Reservasi Servis — Cars City](#3-reservasi-servis--cars-city)
4. [Basis Data Sparepart — Sakura Korea Parts](#4-basis-data-sparepart--sakura-korea-parts)
5. [Computer Vision — Vehicle & PPE Detection](#5-computer-vision--vehicle--ppe-detection)

---

## 1. Sistem Inventaris — Madewi Cookies

**Stack:** Laravel 10 · MySQL · Bootstrap 5 · WhatsApp API · Google Maps API

### 1.1 Mekanisme Pengurangan Stok Otomatis Berbasis Resep (*Recipe-Based Auto Deduction*)

Sistem ini menerapkan prinsip **Bill of Materials (BOM)** yang diadaptasi ke industri kuliner. Setiap produk cookies memiliki entitas `Recipe` yang merepresentasikan komposisi bahan baku beserta kuantitasnya dalam satuan terkecil (gram/ml).

**Arsitektur Relasional:**

```
products ──┐
            ├── recipes (pivot) ── raw_materials
sales ──────┘
```

- Tabel `recipes` berfungsi sebagai **junction table** yang memetakan relasi *many-to-many* antara `products` dan `raw_materials`, menyimpan kolom `quantity_needed` per unit produk.
- Tabel `raw_materials` menyimpan `current_stock` sebagai sumber kebenaran tunggal (*single source of truth*) untuk saldo stok.

**Alur Transaksi — Database Transaction dengan Atomicity:**

```
┌─────────────────────────────────────────────────────────┐
│  CLIENT: Submit Penjualan (product_id, qty_sold)        │
└──────────────────────┬──────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────┐
│  DB::beginTransaction()                                 │
│                                                         │
│  1. INSERT INTO sales (product_id, quantity, total)      │
│                                                         │
│  2. FOREACH recipe WHERE product_id = ?:                │
│     ┌───────────────────────────────────────────────┐   │
│     │ deduction = qty_sold × recipe.quantity_needed  │   │
│     │                                               │   │
│     │ IF raw_material.current_stock < deduction:    │   │
│     │    → ROLLBACK + throw InsufficientStockException│  │
│     │ ELSE:                                         │   │
│     │    → UPDATE raw_materials                     │   │
│     │      SET current_stock = current_stock - deduction│ │
│     │      WHERE id = recipe.raw_material_id        │   │
│     └───────────────────────────────────────────────┘   │
│                                                         │
│  3. CHECK semua raw_materials terkait:                   │
│     IF current_stock <= minimum_threshold:               │
│        → DISPATCH WhatsAppAlertJob (async)               │
│                                                         │
│  DB::commit()                                           │
└─────────────────────────────────────────────────────────┘
```

Seluruh operasi dibungkus dalam `DB::transaction()` untuk menjamin **ACID Compliance** — jika salah satu bahan baku tidak mencukupi, seluruh transaksi di-*rollback* tanpa meninggalkan data inkonsisten (*dirty state*). Pendekatan **Pessimistic Locking** (`lockForUpdate()`) diterapkan pada baris `raw_materials` untuk mencegah *race condition* pada transaksi konkuren.

**Contoh Kalkulasi:**
Jika "Paket Cookies A" membutuhkan 200g Tepung + 100g Mentega + 50g Gula, dan pelanggan membeli 3 paket:
- Tepung: `current_stock -= 3 × 200 = 600g`
- Mentega: `current_stock -= 3 × 100 = 300g`
- Gula: `current_stock -= 3 × 50 = 150g`

### 1.2 Integrasi WhatsApp API — Asynchronous Real-Time Alert

Sistem notifikasi menggunakan **Event-Driven Architecture** dengan Laravel Queue untuk memastikan proses pengiriman pesan tidak memblokir alur transaksi utama (*non-blocking I/O*).

**Arsitektur Notifikasi:**

```
[Stock Update] → Event: LowStockDetected
                     │
                     ▼
              Listener: SendWhatsAppAlert
                     │
                     ▼
              Queue Job (async) → WhatsApp Gateway API
                     │
                     ▼
              Admin menerima pesan:
              "⚠️ PERINGATAN STOK KRITIS
               Bahan: Tepung Terigu
               Sisa: 2.5 kg (Batas Min: 5 kg)
               Segera lakukan restocking."
```

**Mekanisme Threshold:**
- Setiap `raw_material` memiliki kolom `minimum_threshold` yang ditentukan oleh admin.
- Setelah setiap transaksi yang mengurangi stok, sistem menjalankan pengecekan: `current_stock <= minimum_threshold`.
- Jika kondisi terpenuhi, `LowStockDetected` event di-*dispatch* dan diproses secara **asynchronous** melalui Laravel Queue Worker.
- Pendekatan ini menerapkan prinsip **Separation of Concerns** — logika bisnis transaksi terpisah dari logika notifikasi, sehingga kegagalan pengiriman WhatsApp tidak mempengaruhi integritas data penjualan.

### 1.3 Google Maps API — Geolocation & Store Locator

Google Maps JavaScript API diintegrasikan pada halaman publik (*landing page*) untuk menampilkan **marker interaktif** lokasi fisik toko Madewi Cookies.

**Implementasi Teknis:**
- Koordinat latitude/longitude toko disimpan sebagai konfigurasi pada environment variable (`STORE_LAT`, `STORE_LNG`).
- Maps dirender secara *client-side* menggunakan `google.maps.Map` dengan `InfoWindow` yang menampilkan nama toko, alamat lengkap, dan tombol navigasi (*"Get Directions"*).
- Penerapan **lazy loading** pada script Maps API untuk mengoptimalkan *First Contentful Paint (FCP)* halaman.

---

## 2. Manajemen IKM — Kurnia Jati Furniture

**Stack:** Laravel 10 · MySQL · Bootstrap 5 · Eloquent ORM

### 2.1 Arsitektur Produksi Terpadu (*Integrated Production Architecture*)

Sistem ini dirancang untuk mendigitalkan seluruh siklus operasional IKM furnitur — dari penerimaan pesanan kustom hingga pelacakan status pengiriman. Arsitektur mengikuti pola **Domain-Driven Design (DDD)** dengan pemisahan modul yang jelas.

**Entity Relationship — Core Domain:**

```
customers ──── orders ──── order_details ──── products
                  │
                  ├── production_tracking
                  │
                  └── documents (PKB/Surat Jalan)
```

### 2.2 Manajemen Pesanan Kustom (*Custom Order Pipeline*)

Berbeda dengan sistem e-commerce standar, furnitur kustom memiliki variasi dimensi, material, dan finishing yang unik per pesanan. Sistem menangani kompleksitas ini melalui skema **Polymorphic Order Details**.

**State Machine — Order Lifecycle:**

```
[PENDING] → [CONFIRMED] → [IN_PRODUCTION] → [QUALITY_CHECK] → [READY] → [DELIVERED]
    │            │               │                │               │
    └── CANCEL   └── REVISION    └── HOLD         └── REWORK      └── COMPLAINT
```

Setiap transisi status divalidasi menggunakan **State Pattern** — misalnya, status tidak bisa langsung loncat dari `PENDING` ke `DELIVERED` tanpa melewati fase produksi. Setiap perubahan status tercatat dalam tabel `order_status_logs` sebagai **audit trail** untuk keperluan akuntabilitas.

**Logika Validasi Transisi:**

```php
// Aturan transisi yang diizinkan
$allowedTransitions = [
    'PENDING'       => ['CONFIRMED', 'CANCEL'],
    'CONFIRMED'     => ['IN_PRODUCTION', 'REVISION', 'CANCEL'],
    'IN_PRODUCTION' => ['QUALITY_CHECK', 'HOLD'],
    'QUALITY_CHECK' => ['READY', 'REWORK'],
    'READY'         => ['DELIVERED'],
];

// Validasi sebelum update
if (!in_array($newStatus, $allowedTransitions[$currentStatus])) {
    throw new InvalidTransitionException();
}
```

### 2.3 Relasi Data Pelanggan-Pesanan (*Relational Integrity*)

Sistem menerapkan **Database Normalization** hingga bentuk normal ketiga (3NF) untuk menghilangkan redundansi data:

| Layer | Tabel | Fungsi |
|-------|-------|--------|
| Master | `customers` | Data pelanggan (nama, telepon, alamat) |
| Transaction | `orders` | Header pesanan (tanggal, status, total) |
| Detail | `order_details` | Item pesanan (produk, qty, spesifikasi kustom) |
| Reference | `products` | Katalog produk standar + harga dasar |

**Eloquent Relationship Chain:**

```
Customer::with(['orders.details.product'])->find($id);
```

Query ini menggunakan **Eager Loading** untuk mengambil seluruh hierarki data dalam maksimal 4 query SQL (bukan N+1 query), memastikan performa tetap optimal meskipun pelanggan memiliki riwayat pesanan yang banyak.

### 2.4 Dokumen PKB (*Production Work Order*)

Sistem secara otomatis men-*generate* dokumen Perintah Kerja Bengkel (PKB) dalam format yang siap cetak setelah pesanan dikonfirmasi. Dokumen ini berisi spesifikasi teknis furnitur yang menjadi panduan bagi tim produksi, mengurangi risiko miskomunikasi antara bagian penjualan dan workshop.

---

## 3. Reservasi Servis — Cars City

**Stack:** Laravel 10 · MySQL · Bootstrap 5 · WhatsApp Gateway · Google Maps API

### 3.1 Algoritma Anti-Bentrok (*Collision Prevention Algorithm*)

Inti dari sistem reservasi adalah **Temporal Slot Validation** — algoritma yang memastikan tidak ada dua reservasi yang menggunakan teknisi yang sama pada waktu yang bertumpukan (*overlapping*).

**Skema Database Pendukung:**

```
technicians ──── reservations ──── services
                      │
                      ├── reservation_date
                      ├── start_time
                      ├── end_time (calculated)
                      └── status
```

**Algoritma Pengecekan Ketersediaan:**

```
┌──────────────────────────────────────────────────────┐
│  INPUT: technician_id, requested_date, requested_time│
│         service_id (untuk kalkulasi durasi)           │
└──────────────────────┬───────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────┐
│  1. Hitung end_time:                                 │
│     end_time = requested_time + service.duration     │
│                                                      │
│  2. Query konflik (Overlap Detection):               │
│     SELECT COUNT(*) FROM reservations                │
│     WHERE technician_id = ?                          │
│       AND reservation_date = ?                       │
│       AND status NOT IN ('CANCELLED', 'COMPLETED')   │
│       AND (                                          │
│         (start_time < ? AND end_time > ?)            │
│         -- existing reservation overlaps new one     │
│       )                                              │
│     [params: calculated_end, requested_start]        │
│                                                      │
│  3. IF count > 0:                                    │
│     → Return CONFLICT + suggest available slots      │
│     ELSE:                                            │
│     → INSERT reservation + DISPATCH confirmation     │
└──────────────────────────────────────────────────────┘
```

**Visualisasi Deteksi Overlap:**

```
Timeline Teknisi A (Hari X):
 08:00    09:00    10:00    11:00    12:00
   |========|  Reservasi #1 (Servis Ringan)
                  |==============|  Reservasi #2 (Tune-Up)
        |=====|  ← Request Baru: DITOLAK (overlap #1)
                              |====|  ← Request Baru: DITOLAK (overlap #2)
                                       |====|  ← Request Baru: DITERIMA ✓
```

Kondisi overlap terjadi ketika: `new_start < existing_end AND new_end > existing_start`. Rumus ini menangkap semua kemungkinan tumpang tindih — baik sebagian maupun seluruhnya.

### 3.2 Google Maps API — Lokasi Bengkel & Navigasi

Integrasi Maps pada sistem Cars City memiliki dua fungsi:

1. **Store Locator (Publik):** Menampilkan pin lokasi bengkel pada peta interaktif di halaman reservasi, lengkap dengan `InfoWindow` berisi jam operasional dan nomor telepon.
2. **Direction Service:** Tombol *"Navigasi ke Bengkel"* memanfaatkan `google.maps.DirectionsService` untuk menampilkan rute dari lokasi pengguna saat ini ke bengkel, membantu pelanggan yang belum familiar dengan area sekitar.

### 3.3 WhatsApp Gateway — Bukti Konfirmasi Otomatis

Setelah reservasi berhasil divalidasi dan disimpan, sistem men-*dispatch* **Asynchronous Notification Job** untuk mengirim bukti konfirmasi via WhatsApp.

**Payload Pesan Konfirmasi:**

```
✅ KONFIRMASI RESERVASI BERHASIL
━━━━━━━━━━━━━━━━━━━━━━━
No. Reservasi : #RES0619
Nama          : [Nama Pelanggan]
Kendaraan     : [Merk] - [No. Polisi]
Layanan       : [Jenis Servis]
Tanggal       : [DD/MM/YYYY]
Waktu         : [HH:MM] WIB
Teknisi       : [Nama Teknisi]
━━━━━━━━━━━━━━━━━━━━━━━
Harap datang 10 menit sebelum jadwal.
Bengkel Cars City - [Alamat]
```

Penggunaan **queue-based delivery** memastikan respons HTTP ke pelanggan tetap cepat (< 500ms), sementara proses pengiriman WhatsApp berjalan di *background* tanpa memblokir user experience.

---

## 4. Basis Data Sparepart — Sakura Korea Parts

**Stack:** MySQL · Database Design · SQL Optimization

### 4.1 Strategi Perancangan Skema — High-Volume Transaction Database

Sistem dirancang untuk menangani **ribuan SKU (Stock Keeping Unit)** sparepart dengan volume transaksi harian yang tinggi. Pendekatan arsitektural mengikuti prinsip **Database Normalization (3NF)** dengan pertimbangan *selective denormalization* pada titik-titik yang membutuhkan performa baca tinggi.

**Entity Relationship Diagram:**

```
suppliers ──── purchase_orders ──── purchase_details ──── spareparts
                                                              │
customers ──── sales_orders ────── sales_details ─────────────┘
                                                              │
                                                         categories
                                                              │
                                                         inventory_logs
```

**Strategi Normalisasi:**

| Bentuk Normal | Implementasi |
|---------------|-------------|
| 1NF | Setiap kolom bersifat atomik — tidak ada multi-value (misal: kategori tidak disimpan sebagai CSV) |
| 2NF | Eliminasi *partial dependency* — `purchase_details` memisahkan detail item dari header PO |
| 3NF | Eliminasi *transitive dependency* — `category_name` hanya ada di tabel `categories`, tidak duplikasi di `spareparts` |

### 4.2 Database Indexing — Optimasi Query untuk Pencarian Instan

Pada dataset dengan ribuan baris, pencarian tanpa indeks menghasilkan **Full Table Scan** yang semakin lambat seiring pertumbuhan data. Strategi indexing diterapkan secara selektif berdasarkan pola akses data:

**Strategi Indexing:**

```sql
-- 1. Primary Index (otomatis pada PK)
-- Setiap tabel menggunakan AUTO_INCREMENT id sebagai clustered index

-- 2. Search Index — kolom yang sering digunakan untuk pencarian
CREATE INDEX idx_spareparts_part_number ON spareparts(part_number);
CREATE INDEX idx_spareparts_name ON spareparts(name);

-- 3. Filter Index — kolom yang sering digunakan di WHERE clause
CREATE INDEX idx_spareparts_category ON spareparts(category_id);
CREATE INDEX idx_inventory_date ON inventory_logs(transaction_date);

-- 4. Composite Index — untuk query multi-kondisi
CREATE INDEX idx_sales_customer_date 
    ON sales_orders(customer_id, order_date);
```

**Dampak Performa:**

```
TANPA INDEX (Full Table Scan):
┌─────────────────────────────────────────┐
│ SELECT * FROM spareparts                │
│ WHERE part_number = 'HND-BRK-0042'     │
│                                         │
│ Rows Examined: 15,000 → Time: ~120ms   │
└─────────────────────────────────────────┘

DENGAN INDEX (Index Seek):
┌─────────────────────────────────────────┐
│ SELECT * FROM spareparts                │
│ WHERE part_number = 'HND-BRK-0042'     │
│                                         │
│ Rows Examined: 1 → Time: ~0.5ms        │
└─────────────────────────────────────────┘
```

### 4.3 Optimasi Query — Menghindari Anti-Pattern

```sql
-- ❌ ANTI-PATTERN: N+1 Query (mengambil detail di loop)
SELECT * FROM sales_orders;
-- Lalu untuk SETIAP order:
SELECT * FROM sales_details WHERE order_id = ?;
-- Total query: 1 + N (jika 100 order = 101 query!)

-- ✅ OPTIMIZED: JOIN dalam satu query
SELECT so.*, sd.sparepart_id, sd.quantity, sd.unit_price,
       sp.name AS sparepart_name
FROM sales_orders so
JOIN sales_details sd ON so.id = sd.order_id
JOIN spareparts sp ON sd.sparepart_id = sp.id
WHERE so.order_date BETWEEN '2026-01-01' AND '2026-01-31';
-- Total query: 1 (dengan indeks, eksekusi < 5ms)
```

### 4.4 Integritas Referensial (*Referential Integrity*)

Semua relasi antar tabel dijaga melalui **Foreign Key Constraints** dengan strategi `ON DELETE` yang tepat:

```sql
-- Mencegah penghapusan sparepart yang masih memiliki transaksi
ALTER TABLE sales_details 
    ADD CONSTRAINT fk_sales_sparepart
    FOREIGN KEY (sparepart_id) REFERENCES spareparts(id)
    ON DELETE RESTRICT;

-- Cascade delete untuk detail ketika header order dihapus
ALTER TABLE purchase_details
    ADD CONSTRAINT fk_purchase_detail_order
    FOREIGN KEY (purchase_order_id) REFERENCES purchase_orders(id)
    ON DELETE CASCADE;
```

---

## 5. Computer Vision — Vehicle & PPE Detection

**Stack:** Python · YOLOv8 · OpenCV · Flask/Streamlit · NumPy

### 5.1 Arsitektur Pipeline Deteksi (*Detection Pipeline Architecture*)

Sistem menggunakan model **YOLOv8 (You Only Look Once v8)** dari Ultralytics sebagai *inference engine* utama. YOLO dipilih karena kemampuannya melakukan **single-pass detection** — mendeteksi dan mengklasifikasikan objek dalam satu kali forward pass jaringan neural, menghasilkan latensi rendah yang cocok untuk aplikasi *real-time*.

**End-to-End Pipeline:**

```
┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  INPUT   │    │ PREPROCESSING│    │  INFERENCE    │    │ POSTPROCESS  │
│          │───▶│              │───▶│              │───▶│              │
│ Gambar/  │    │ • Resize     │    │ • YOLOv8     │    │ • NMS Filter │
│ Video/   │    │   (640×640)  │    │   Forward    │    │ • Confidence │
│ Webcam   │    │ • Normalize  │    │   Pass       │    │   Threshold  │
│          │    │   (0-1)      │    │ • Tensor     │    │ • BBox Draw  │
│          │    │ • RGB Convert│    │   Output     │    │ • Label Map  │
└──────────┘    └──────────────┘    └──────────────┘    └──────┬───────┘
                                                               │
                                                               ▼
                                                    ┌──────────────────┐
                                                    │  OUTPUT          │
                                                    │ • Annotated Image│
                                                    │ • JSON Data:     │
                                                    │   [{class, conf, │
                                                    │     bbox}]       │
                                                    │ • Business Logic │
                                                    └──────────────────┘
```

**Detail Setiap Tahap:**

1. **Preprocessing:** Citra input di-*resize* ke resolusi 640×640 piksel (input standard YOLOv8), dinormalisasi dari rentang [0-255] ke [0.0-1.0], dan dikonversi dari BGR (format OpenCV) ke RGB.

2. **Inference:** Model YOLOv8 yang telah di-*fine-tune* pada dataset spesifik (kendaraan/PPE) melakukan forward pass. Output berupa tensor yang berisi *bounding box coordinates* (x, y, w, h), *confidence score*, dan *class probability* untuk setiap objek terdeteksi.

3. **Postprocessing:** **Non-Maximum Suppression (NMS)** diterapkan untuk mengeliminasi *duplicate detections* pada objek yang sama. Hanya deteksi dengan confidence ≥ threshold (default: 0.5) yang dipertahankan.

### 5.2 Deteksi Kendaraan Tol — Klasifikasi Golongan & Kalkulasi Tarif

Modul ini mengkonversi output deteksi visual menjadi **keputusan bisnis otomatis** — tarif tol yang harus dibayar.

**Alur Konversi AI-to-Business:**

```
┌───────────────────────────────────────────────────────────┐
│  TAHAP 1: DETEKSI & KLASIFIKASI                          │
│                                                           │
│  YOLOv8 mendeteksi kendaraan dan mengklasifikasikan:      │
│  ┌─────────────┬──────────────────────────────────┐       │
│  │ Class ID    │ Kelas Kendaraan                  │       │
│  ├─────────────┼──────────────────────────────────┤       │
│  │ 0           │ Golongan I (Sedan, Jeep, Pickup) │       │
│  │ 1           │ Golongan II (Truk 2 sumbu)       │       │
│  │ 2           │ Golongan III (Truk 3 sumbu)      │       │
│  │ 3           │ Golongan IV (Truk 4 sumbu)       │       │
│  │ 4           │ Golongan V (Truk 5+ sumbu)       │       │
│  └─────────────┴──────────────────────────────────┘       │
│                                                           │
│  TAHAP 2: MAPPING KE TARIF                                │
│                                                           │
│  tariff_table = {                                         │
│      "Golongan I"   : Rp 16.000,                         │
│      "Golongan II"  : Rp 16.000,                         │
│      "Golongan III" : Rp 16.500,                         │
│      "Golongan IV"  : Rp 22.000,                         │
│      "Golongan V"   : Rp 28.500,                         │
│  }                                                        │
│                                                           │
│  TAHAP 3: OUTPUT DIGITAL                                  │
│                                                           │
│  {                                                        │
│    "vehicle_detected": true,                              │
│    "class": "Golongan III",                               │
│    "confidence": 0.92,                                    │
│    "tariff": 16500,                                       │
│    "timestamp": "2026-05-04T10:30:15",                    │
│    "bbox": [120, 80, 450, 320]                            │
│  }                                                        │
└───────────────────────────────────────────────────────────┘
```

**Logika Kalkulasi Real-Time:**

```python
# Pseudocode — Inference + Business Logic
results = model.predict(frame, conf=0.5)

for detection in results[0].boxes:
    class_id = int(detection.cls)
    confidence = float(detection.conf)
    class_name = CLASS_MAPPING[class_id]     # "Golongan III"
    tariff = TARIFF_TABLE[class_name]        # Rp 16.500

    # Render pada frame
    annotate_frame(frame, detection.xyxy, 
                   f"{class_name} | Rp {tariff:,} | {confidence:.0%}")

    # Simpan ke database/log
    log_detection(class_name, tariff, confidence, timestamp)
```

### 5.3 Deteksi PPE — Kepatuhan Keselamatan Kerja

Modul PPE Detection mengidentifikasi apakah pekerja di area industri mengenakan Alat Pelindung Diri (APD) yang sesuai standar.

**Kelas Deteksi:**

| Class | Objek | Status |
|-------|-------|--------|
| 0 | Helm (Hardhat) | ✅ Compliant |
| 1 | Rompi (Safety Vest) | ✅ Compliant |
| 2 | Tanpa Helm | ⚠️ Violation |
| 3 | Tanpa Rompi | ⚠️ Violation |

**Logika Compliance Check:**

```python
violations = []
for detection in results:
    if detection.class_name in ['NO-Helmet', 'NO-Vest']:
        violations.append({
            'type': detection.class_name,
            'confidence': detection.conf,
            'location': detection.xyxy,
            'timestamp': datetime.now()
        })

if violations:
    trigger_safety_alert(violations)
    save_violation_log(violations)
```

Sistem ini mengubah output visual model AI menjadi **actionable safety data** — memungkinkan manajer K3 (Keselamatan dan Kesehatan Kerja) untuk memonitor kepatuhan APD secara *real-time* tanpa harus melakukan inspeksi manual secara konstan.

---

## Rangkuman Kompetensi Teknis

| Aspek | Implementasi |
|-------|-------------|
| **Database Design** | Normalisasi 3NF, Foreign Key Constraints, Composite Indexing |
| **Backend Logic** | ACID Transaction, State Machine, Pessimistic Locking |
| **API Integration** | WhatsApp Gateway, Google Maps JavaScript API |
| **Architecture Pattern** | Event-Driven, Queue-Based Async, Separation of Concerns |
| **AI/ML Pipeline** | YOLOv8 Inference, NMS, Confidence Thresholding |
| **Business Logic** | BOM-based Deduction, Temporal Slot Validation, AI-to-Business Mapping |

---

*Dokumentasi ini disusun untuk mendemonstrasikan kemampuan Logical Thinking dan System Design dalam konteks pengembangan aplikasi nyata.*

**© 2026 Feyza Revalina** · [GitHub](https://github.com/feyfryvzn) · [Portfolio](https://github.com/feyfryvzn/Portfolio)
