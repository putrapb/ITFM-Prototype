# e-Procurement BNI

Sistem pengajuan dan persetujuan pengadaan (e-Procurement) berbasis **Laravel 13**, **Tailwind CSS**, dan **Alpine.js** untuk lingkungan **IT Infrastructure Management**. Aplikasi ini mengelola alur tiket pengadaan dari pengajuan requester, pemeriksaan dokumen oleh Team Leader, validasi otomatis (Smart Validation 4-Gate), keputusan Department Head, hingga penerbitan Form Pengadaan (PDF) dan arsip audit.

---

## Daftar Isi

- [Fitur Utama](#fitur-utama)
- [Peran Pengguna](#peran-pengguna)
- [Alur Tiket](#alur-tiket)
- [Smart Validation 4-Gate](#smart-validation-4-gate)
- [Tech Stack](#tech-stack)
- [Kebutuhan Sistem](#kebutuhan-sistem)
- [Instalasi](#instalasi)
- [Konfigurasi Environment](#konfigurasi-environment)
- [Menjalankan Aplikasi](#menjalankan-aplikasi)
- [Akun Demo](#akun-demo)
- [Pengujian](#pengujian)
- [Struktur Direktori](#struktur-direktori)
- [Deployment](#deployment)
- [Catatan Keamanan](#catatan-keamanan)

---

## Fitur Utama

- **Autentikasi terintegrasi HR** — registrasi divalidasi terhadap data karyawan (NIP), pembatasan divisi, dan verifikasi **OTP via email**.
- **Role-Based Access Control (RBAC)** — tiga peran: `requester`, `team_leader`, `department_head`.
- **Multi-item pengajuan** — satu tiket dapat berisi banyak item dengan kuantitas dan harga satuan.
- **Smart Validation 4-Gate** — deteksi duplikasi, validasi nominal, saran klasifikasi CAPEX/OPEX, dan pengecekan ketersediaan anggaran.
- **Klasifikasi CAPEX/OPEX fleksibel** — berdasarkan kategori, kata kunci judul/deskripsi, dan dapat dioverride per item.
- **Manajemen anggaran** — pagu per kategori per jenis pengeluaran per tahun fiskal (`total_limit`, `locked_amount`, `used_amount`), termasuk **silang dana (cross-fund)** saat anggaran tidak cukup.
- **Workflow persetujuan berjenjang** — review dokumen per file, keputusan approve/decline, serta aksi **bulk** untuk Team Leader dan Department Head.
- **Upload & verifikasi dokumen** — dukungan banyak dokumen, status per dokumen (accepted/rejected/pending), dan feedback revisi.
- **Form Pengadaan PDF** — digenerate dengan DomPDF, dilengkapi barcode/QR dan persetujuan digital signature.
- **Arsip audit (DIP)** — unduh paket audit untuk tiket final dalam format PDF.
- **Notifikasi in-app** — pemberitahuan per peran dan per pengguna.
- **Audit trail immutable** — log persetujuan tidak dapat diubah atau dihapus.
- **Dashboard adaptif per peran** — ringkasan status tiket, tren bulanan, dan utilisasi anggaran (Chart.js).
- **Ekspor CSV** — unduh daftar tiket sesuai filter.
- **Verifikasi publik via QR** — halaman verifikasi tiket dengan signed URL.

---

## Peran Pengguna

| Peran | Deskripsi | Kemampuan Utama |
|---|---|---|
| `requester` | Staf IT Infrastructure Project Management | Membuat, mengedit, merevisi, membatalkan tiket; menjalankan Smart Validation; mengajukan silang dana |
| `team_leader` | Pemeriksa dokumen & penerbit form | Review dokumen (per file / bulk), meneruskan ke Department Head, menerbitkan Form Pengadaan PDF |
| `department_head` | Pengambil keputusan akhir | Menyetujui / menolak tiket (per tiket / bulk), melihat log audit |

Peran ditentukan otomatis saat registrasi berdasarkan jabatan pada data HR (`HrEmployee::deriveRole()`).

---

## Alur Tiket

```
Requester          Team Leader            Department Head         Team Leader
    │                   │                       │                     │
    │  Ajukan tiket     │                       │                     │
    ├──────────────────▶│ pending_review        │                     │
    │                   │                       │                     │
    │   (Revisi)        │  Tolak dokumen        │                     │
    │◀──────────────────┤ revision              │                     │
    │                   │                       │                     │
    │                   │  Terima dokumen       │                     │
    │                   ├──────────────────────▶│ pending_dept_head   │
    │  Smart Validation │                       │                     │
    │  (4-Gate)         │                       │                     │
    │                   │                       │  Approve            │
    │                   │                       ├────────────────────▶│ approved
    │                   │                       │                     │
    │                   │                       │  Decline            │
    │                   │                       ├──▶ declined         │
    │                   │                       │                     │
    │                   │                       │      Terbitkan Form │
    │◀──────────────────┴───────────────────────┴─────────────────────┤ form_generated
```

**Status tiket:**

| Status | Label |
|---|---|
| `pending_review` | Menunggu Cek Dokumen |
| `revision` | Revisi Dokumen |
| `need_to_validate` | Perlu Validasi |
| `pending_dept_head` | Menunggu Dept Head |
| `approved` | Disetujui |
| `declined` | Ditolak |
| `form_generated` | Form Diterbitkan |

---

## Smart Validation 4-Gate

Dijalankan oleh requester sebelum tiket diteruskan ke Department Head. Setiap gate hanya berjalan bila gate sebelumnya lolos.

| Gate | Pemeriksaan | Sifat |
|---|---|---|
| **Gate 1** | Deteksi duplikasi judul tiket aktif milik requester yang sama | Peringatan lunak (dapat dikonfirmasi) |
| **Gate 2** | Validasi nominal: gagal jika ≤ Rp 0; peringatan jika > Rp 99 Miliar | Hard fail / peringatan lunak |
| **Gate 3** | Saran klasifikasi CAPEX/OPEX berdasarkan kata kunci & kategori | Rekomendasi (dapat dioverride) |
| **Gate 4** | Ketersediaan saldo anggaran dan batas bulanan (≤ 30% dari pagu bulanan) | Keras (memblokir jika tidak cukup) |

Jika Gate 4 gagal, requester dapat mengajukan **silang dana** ke pos anggaran alternatif (CAPEX ↔ OPEX).

Aturan kata kunci klasifikasi dapat dikonfigurasi di `config/eprocurement.php` (`classification_rules`).

---

## Tech Stack

| Komponen | Teknologi |
|---|---|
| Backend | PHP 8.3, Laravel 13 |
| Frontend | Blade, Tailwind CSS 3/4, Alpine.js, Vite |
| Database | PostgreSQL (Supabase) / SQLite untuk testing |
| PDF | barryvdh/laravel-dompdf, milon/barcode |
| Testing | Pest PHP 4, PHPUnit 12 |
| Code Style | Laravel Pint |

---

## Kebutuhan Sistem

- PHP **8.3+** dengan ekstensi: `pdo`, `mbstring`, `openssl`, `tokenizer`, `xml`, `ctype`, `json`, `bcmath`, `fileinfo`, `gd`
- Composer 2
- Node.js 18+ dan npm
- PostgreSQL 14+ (produksi) atau SQLite (lokal/testing)

---

## Instalasi

```bash
# 1. Clone repository
git clone https://github.com/putrapb/ITFM-Prototype.git
cd ITFM-Prototype

# 2. Install dependency PHP
composer install

# 3. Salin environment & generate key
cp .env.example .env
php artisan key:generate

# 4. Sesuaikan koneksi database pada .env
#    (lihat bagian Konfigurasi Environment)

# 5. Jalankan migrasi & seeder
php artisan migrate --seed

# 6. Install dependency frontend & build asset
npm install
npm run build

# 7. Buat symbolic link storage
php artisan storage:link

# 8. Jalankan server
php artisan serve
```

Atau gunakan script bawaan Composer:

```bash
composer run setup
```

---

## Konfigurasi Environment

Variabel penting pada `.env`:

```env
APP_NAME="e-Procurement BNI"
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost

# Database (contoh PostgreSQL / Supabase)
DB_CONNECTION=pgsql
DB_HOST=aws-1-ap-southeast-2.pooler.supabase.com
DB_PORT=5432
DB_DATABASE=postgres
DB_USERNAME=postgres.<project-ref>
DB_PASSWORD=
# DB_PERSISTENT=true (untuk connection pooler Supabase)

# Email (digunakan untuk OTP & notifikasi)
MAIL_MAILER=smtp
MAIL_HOST=smtp.example.com
MAIL_PORT=587
MAIL_USERNAME=
MAIL_PASSWORD=
MAIL_FROM_ADDRESS="eprocurement@bni.co.id"

# Opsi e-Procurement
CAPEX_THRESHOLD=200000000
OTP_TTL_MINUTES=10
ALLOWED_DIVISION="IT Infrastructure Management"
DEPLOY_SECRET=
```

> **Catatan:** Jangan pernah commit file `.env` yang berisi kredensial asli. Gunakan `.env.example` sebagai template.

---

## Menjalankan Aplikasi

**Development** (server + queue + log + Vite sekaligus):

```bash
composer run dev
```

Atau secara terpisah:

```bash
php artisan serve
npm run dev
```

**Build produksi:**

```bash
npm run build
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

---

## Akun Demo

Setelah menjalankan `php artisan db:seed` (menggunakan `UserSeeder`), tersedia akun berikut. Semua password: `password`.

| Peran | Email | Nama |
|---|---|---|
| Requester | `raihan@bna.co.id` | Raihan Ardiansyah |
| Team Leader | `haikal@bna.co.id` | Haikal Fadhilah |
| Department Head | `putra@bna.co.id` | Putra Bagas P. |

Seeder lain yang tersedia: `HrEmployeeSeeder`, `BudgetSeeder`, `TicketTestSeeder`, `UtDataSeeder`, `RecalculateBudgetSeeder`.

---

## Pengujian

Project menggunakan **Pest PHP**.

```bash
# Jalankan seluruh test
php artisan test

# atau
composer run test

# Test tertentu
php artisan test --filter=SmartValidationTest
```

Cakupan test mencakup: lifecycle tiket, rantai persetujuan, RBAC, autentikasi/OTP, reset password, audit log, dan Smart Validation.

---

## Struktur Direktori

```
app/
├── Console/Commands/       # Command kustom (migrasi item legacy)
├── Http/
│   ├── Controllers/        # Controller utama & autentikasi
│   ├── Middleware/         # RBAC, security headers, verifikasi deploy
│   └── Requests/           # Form request validation
├── Mail/                   # Mailable OTP
├── Models/                 # Ticket, Budget, User, ApprovalLog, dll.
└── Services/               # SmartValidationService

config/
└── eprocurement.php        # Kategori, aturan klasifikasi, OTP, storage

database/
├── factories/              # Factory untuk testing
├── migrations/             # Skema database
└── seeders/                # Data awal & demo

resources/views/
├── auth/                   # Login, register, OTP, reset password
├── dashboard/              # Dashboard per peran
├── tickets/                # Daftar, detail, create, edit, audit PDF
├── pdf/                    # Template Form Pengadaan
└── settings/, faq/, profile/, audit-logs/

routes/
└── web.php                 # Definisi seluruh rute
```

---

## Deployment

Aplikasi ini dipersiapkan untuk deployment ke **cPanel** melalui:

- **`.cpanel.yml`** — menyalin folder `public/`, menyesuaikan path `index.php`, membuat symlink storage, menjalankan migrasi, dan membersihkan cache.
- **Webhook post-deploy** — `GET /api/deploy/post-update`, dilindungi middleware `VerifyDeploySecret` (header rahasia `DEPLOY_SECRET`). Webhook menjalankan `config:clear`, `view:clear`, dan `route:clear`.

Pastikan environment produksi menetapkan `APP_ENV=production`, `APP_DEBUG=false`, dan `DEPLOY_SECRET` yang kuat.

---

## Catatan Keamanan

- Registrasi hanya untuk email domain korporat dan NIP yang terdaftar di data HR dengan divisi yang diizinkan.
- OTP memiliki TTL (default 10 menit), pembatasan percobaan, dan rate limiting.
- Log persetujuan (`ApprovalLog`) bersifat **immutable** untuk menjaga integritas audit trail.
- Route yang mengubah data dilindungi middleware `auth`, `verified`, dan `role`.
- Verifikasi tiket publik menggunakan **signed URL**.
- Seluruh input divalidasi melalui Form Request / validasi controller.
- Koneksi database produksi menggunakan SSL ke Supabase; simpan kredensial di environment, bukan di source code.

---

## Lisensi

Dirilis di bawah lisensi [MIT](https://opensource.org/licenses/MIT).
