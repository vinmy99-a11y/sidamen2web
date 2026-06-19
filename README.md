<div align="center">

# 🎓 SIDAMEN
### Sistem Data Manajemen Mahasiswa

![Laravel](https://img.shields.io/badge/Laravel-11.x-FF2D20?style=for-the-badge&logo=laravel&logoColor=white)
![TailwindCSS](https://img.shields.io/badge/Tailwind_CSS-3.x-38B2AC?style=for-the-badge&logo=tailwind-css&logoColor=white)
![Alpine.js](https://img.shields.io/badge/Alpine.js-3.x-8BC0D0?style=for-the-badge&logo=alpine.js&logoColor=white)
![Pusher](https://img.shields.io/badge/Pusher-Realtime-300D4F?style=for-the-badge&logo=pusher&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?style=for-the-badge&logo=mysql&logoColor=white)

> Aplikasi manajemen data mahasiswa berbasis web dengan fitur CRUD lengkap, algoritma sorting & searching manual, import/export CSV/JSON, notifikasi email realtime, dan activity log realtime via Pusher WebSocket.

</div>

---

## 📋 Daftar Isi

- [Fitur](#-fitur)
- [Tech Stack](#-tech-stack)
- [Struktur Proyek](#-struktur-proyek)
- [Instalasi](#-instalasi)
- [Konfigurasi](#-konfigurasi)
- [Menjalankan Aplikasi](#-menjalankan-aplikasi)
- [Panduan Penggunaan](#-panduan-penggunaan)
- [API Endpoint](#-api-endpoint)
- [Algoritma](#-algoritma)
- [Kredit](#-kredit)

---

## ✨ Fitur

### 🔐 Autentikasi
- Login admin dengan email & password
- Error handling dengan `try-catch` + notifikasi merah
- Animasi shake pada form saat login gagal
- Remember me & session management

### 📊 CRUD Mahasiswa
- **Create** — Form tambah mahasiswa dengan validasi lengkap
- **Read** — Tabel data dengan pagination (50 data/halaman)
- **Update** — Form edit dengan pre-fill data existing
- **Delete** — Konfirmasi pop-up sebelum hapus permanen

### ✅ Validasi Input
- **NIM** — Regex: hanya angka, maksimal 12 digit
- **Email** — Regex: format `nama@domain.tld`
- **Jurusan** — Dropdown tetap (5 pilihan)
- **Angkatan** — Dropdown tahun 2018–sekarang
- Validasi dijalankan di **frontend (Alpine.js) + backend (FormRequest)**

### 📁 Import & Export
- Import file **CSV** dan **JSON** via drag & drop
- Validasi struktur kolom sebelum upload
- Preview 5 baris pertama sebelum submit
- Export data ke **CSV** (Excel-friendly, BOM UTF-8) dan **JSON**
- Riwayat import tersimpan di database

### 🔍 Algoritma Sorting & Searching
| Fitur | Pilihan |
|---|---|
| Sorting | Bubble Sort, Selection Sort, Shell Sort |
| Searching | Linear Search, Binary Search |
| Info | Waktu eksekusi (detik), Notasi Big O |
| Log | Build Line Log — riwayat baris yang diperiksa |

### 📧 Notifikasi Email Realtime
- Email otomatis dikirim ke mahasiswa saat data **dibuat** dan **diubah**
- Template HTML responsif
- Menggunakan **Laravel Queue** (database driver) agar tidak blocking
- Powered by **SMTP Gmail**

### 📡 Activity Log Realtime
- Setiap aksi admin (Create, Update, Delete, Import, Export, Login) tercatat otomatis
- Update **realtime tanpa refresh** via **Pusher WebSocket**
- Filter log berdasarkan tipe aksi
- Tampilkan IP address, timestamp, dan nama admin

### 💬 Komentar Pasca Pencarian
- Tombol komentar muncul otomatis pada data yang ditemukan saat searching
- Modal komentar menampilkan riwayat komentar sebelumnya
- Submit komentar tanpa reload halaman via `fetch()`

### 🔔 Sistem Notifikasi Global
- Toast notification berwarna (hijau = sukses, merah = error)
- Auto-dismiss setelah 4 detik
- Dapat dipanggil dari mana saja via `window.$toast()`

---

## 🛠 Tech Stack

| Layer | Teknologi |
|---|---|
| Framework Backend | Laravel 11 |
| Template Engine | Blade |
| Frontend Reaktif | Alpine.js 3.x |
| CSS Framework | Tailwind CSS 3.x (CDN) |
| Realtime | Pusher + Laravel Echo |
| Database | MySQL / MariaDB |
| Email | SMTP Gmail + Laravel Queue |
| Font | Plus Jakarta Sans, Inter, JetBrains Mono |

---

## 📁 Struktur Proyek

```
sidamen/
├── app/
│   ├── Events/
│   │   └── ActivityLogCreated.php       ← Pusher broadcast event
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── AuthController.php       ← Login, logout, dashboard
│   │   │   ├── MahasiswaController.php  ← CRUD + komentar
│   │   │   ├── ImportExportController.php
│   │   │   └── ActivityLogController.php
│   │   └── Requests/
│   │       ├── StoreMahasiswaRequest.php
│   │       └── UpdateMahasiswaRequest.php
│   ├── Mail/
│   │   └── MahasiswaNotifikasi.php      ← Queueable mailable
│   ├── Models/
│   │   ├── Mahasiswa.php
│   │   ├── Komentar.php
│   │   ├── ActivityLog.php
│   │   └── ImportLog.php
│   ├── Providers/
│   │   └── EventServiceProvider.php
│   └── Services/
│       └── ActivityLogService.php       ← Log + broadcast helper
│
├── config/
│   └── broadcasting.php
│
├── database/
│   ├── migrations/
│   │   ├── ..._create_mahasiswa_table.php
│   │   ├── ..._create_komentars_table.php
│   │   ├── ..._create_activity_logs_table.php
│   │   └── ..._create_import_logs_table.php
│   └── seeders/
│       └── AdminSeeder.php
│
├── resources/
│   └── views/
│       ├── layouts/app.blade.php        ← Layout utama
│       ├── auth/login.blade.php
│       ├── dashboard.blade.php
│       ├── mahasiswa/
│       │   ├── index.blade.php          ← Tabel + sorting + searching
│       │   ├── create.blade.php
│       │   ├── edit.blade.php
│       │   └── import-export.blade.php
│       ├── activity-log/index.blade.php
│       ├── vendor/pagination/tailwind.blade.php
│       └── emails/
│           └── mahasiswa-notifikasi.blade.php
│
├── routes/
│   └── web.php
│
├── .env
└── .env.example
```

---

## 🚀 Instalasi

### Prasyarat

Pastikan sudah terinstall:
- PHP >= 8.2
- Composer
- MySQL / MariaDB
- XAMPP / Laragon (Windows)

### Langkah Instalasi

**1. Buat project Laravel baru**
```bash
composer create-project laravel/laravel sidamen
cd sidamen
```

**2. Install package Pusher**
```bash
composer require pusher/pusher-php-server
```

**3. Copy file dari zip**

Salin file dari kedua zip (`sidamen-frontend.zip` dan `sidamen-backend.zip`) ke folder project:

| Dari Zip | Copy Ke |
|---|---|
| `resources/views/` | `sidamen/resources/views/` |
| `public/css/` | `sidamen/public/css/` |
| `app/` | `sidamen/app/` |
| `database/` | `sidamen/database/` |
| `routes/web.php` | `sidamen/routes/web.php` |
| `config/broadcasting.php` | `sidamen/config/broadcasting.php` |
| `.env.example` | referensi untuk mengisi `.env` |

**4. Buat database**
```sql
CREATE DATABASE sidamen;
```

---

## ⚙️ Konfigurasi

### `.env` — Wajib Diisi

Buka file `.env` dan isi bagian berikut:

```env
APP_NAME=SIDAMEN
APP_URL=http://localhost:8000

# Database
DB_DATABASE=sidamen
DB_USERNAME=root
DB_PASSWORD=

# Queue
QUEUE_CONNECTION=database

# Pusher (daftar gratis di pusher.com)
BROADCAST_DRIVER=pusher
PUSHER_APP_ID=isi-disini
PUSHER_APP_KEY=isi-disini
PUSHER_APP_SECRET=isi-disini
PUSHER_APP_CLUSTER=ap1

# Gmail (gunakan App Password, bukan password biasa)
MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=emaillo@gmail.com
MAIL_PASSWORD=xxxx-xxxx-xxxx-xxxx
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=emaillo@gmail.com
MAIL_FROM_NAME="SIDAMEN"
```

### Mendapatkan Pusher Credentials

1. Daftar di [pusher.com](https://pusher.com) (gratis)
2. Buat app baru → cluster: **ap1 (Asia)**
3. Buka **App Keys** → copy `app_id`, `key`, `secret`

### Mendapatkan Gmail App Password

1. Aktifkan **2-Step Verification** di Google Account
2. Buka **Security → App Passwords**
3. Generate password untuk **Mail / Windows**
4. Gunakan 16 karakter yang muncul sebagai `MAIL_PASSWORD`

### Daftarkan EventServiceProvider

Buka `bootstrap/providers.php`, tambahkan:
```php
return [
    App\Providers\AppServiceProvider::class,
    App\Providers\EventServiceProvider::class, // ← tambahkan
];
```

---

## ▶️ Menjalankan Aplikasi

### 1. Jalankan Migration & Seeder
```bash
php artisan migrate
php artisan queue:table
php artisan session:table
php artisan migrate
php artisan db:seed --class=AdminSeeder
```

### 2. Buka 3 Terminal Terpisah

**Terminal 1 — Web Server**
```bash
php artisan serve
```

**Terminal 2 — Queue Worker (Email)**
```bash
php artisan queue:work
```

**Terminal 3 — (Opsional) Monitor Queue**
```bash
php artisan queue:listen --tries=3
```

### 3. Akses Aplikasi

Buka browser → [http://localhost:8000](http://localhost:8000)

| Field | Value |
|---|---|
| Email | `admin@sidamen.com` |
| Password | `admin123` |

---

## 📖 Panduan Penggunaan

### Tambah Mahasiswa
1. Klik menu **Data Mahasiswa** di sidebar
2. Klik tombol **Tambah Mahasiswa**
3. Isi form: NIM (maks 12 digit angka), Nama, Email, Jurusan, Angkatan
4. Klik **Simpan Data**
5. Email notifikasi otomatis terkirim ke mahasiswa

### Cari Mahasiswa (Searching)
1. Di halaman Data Mahasiswa, ketik NIM atau Nama di kolom pencarian
2. Pilih algoritma: **Linear Search** atau **Binary Search**
3. Hasil pencarian tampil beserta:
   - ⏱ Waktu eksekusi
   - 📊 Notasi Big O
   - 📋 Build Line Log (baris yang diperiksa)
4. Pada data yang ditemukan, klik tombol 💬 **Komentar** untuk menambah catatan

### Urutkan Data (Sorting)
1. Di toolbar tabel, klik dropdown **Urutkan**
2. Pilih kombinasi field (NIM/Nama) dan algoritma (Bubble/Selection/Shell Sort)
3. Data langsung diurutkan secara client-side dengan info waktu eksekusi

### Import Data
1. Klik menu **Import / Export**
2. Drag & drop file CSV/JSON ke area upload, atau klik untuk browse
3. Preview 5 baris pertama akan tampil otomatis
4. Jika ada error format, sistem akan menampilkan daftar error per baris
5. Klik **Import Data** untuk memproses

### Export Data
1. Klik menu **Import / Export**
2. Pilih format: **Unduh CSV** atau **Unduh JSON**
3. File langsung ter-download

### Activity Log
1. Klik menu **Activity Log** di sidebar
2. Log update otomatis realtime tanpa refresh saat ada aksi admin
3. Filter log berdasarkan tipe aksi (Create/Update/Delete/Import/Export)
4. Baris baru di-highlight biru selama 3 detik

---

## 🔗 API Endpoint

| Method | URL | Keterangan |
|---|---|---|
| GET | `/login` | Halaman login |
| POST | `/login` | Proses login |
| POST | `/logout` | Logout |
| GET | `/dashboard` | Dashboard |
| GET | `/mahasiswa` | Daftar mahasiswa |
| GET | `/mahasiswa/create` | Form tambah |
| POST | `/mahasiswa` | Simpan mahasiswa baru |
| GET | `/mahasiswa/{id}/edit` | Form edit |
| PUT | `/mahasiswa/{id}` | Update mahasiswa |
| DELETE | `/mahasiswa/{id}` | Hapus mahasiswa |
| GET | `/mahasiswa/{id}/komentar` | Get komentar (JSON) |
| POST | `/mahasiswa/{id}/komentar` | Simpan komentar |
| GET | `/mahasiswa/import-export` | Halaman import/export |
| POST | `/mahasiswa/import` | Proses upload import |
| GET | `/mahasiswa/export?format=csv` | Download CSV |
| GET | `/mahasiswa/export?format=json` | Download JSON |
| GET | `/activity-log` | Halaman activity log |

---

## 🧮 Algoritma

### Searching

#### Linear Search — `O(n)`
```
Iterasi setiap elemen dari index 0 sampai n-1.
Cocok untuk pencarian parsial (contains) pada NIM atau Nama.
Menampilkan semua baris yang cocok.
```

#### Binary Search — `O(log n)`
```
Array diurutkan by NIM terlebih dahulu.
Cari di tengah → bandingkan → kurangi separuh ruang pencarian.
Cocok untuk pencarian exact match NIM.
Jauh lebih cepat pada dataset besar.
```

### Sorting

#### Bubble Sort — `O(n²)`
```
Bandingkan elemen berdekatan, tukar jika salah urutan.
Ulangi n-1 pass. Sederhana tapi lambat untuk data besar.
```

#### Selection Sort — `O(n²)`
```
Temukan elemen minimum, pindahkan ke posisi paling kiri.
Ulangi untuk sisa array. Lebih sedikit swap dibanding Bubble Sort.
```

#### Shell Sort — `O(n log n)` rata-rata
```
Pengembangan Insertion Sort dengan gap sequence.
Mulai dengan gap besar, kurangi sampai 1.
Jauh lebih cepat dari Bubble/Selection untuk dataset besar.
```

---

## 👨‍💻 Kredit

Dibangun dengan ❤️ menggunakan:
- [Laravel](https://laravel.com) — The PHP Framework for Web Artisans
- [Tailwind CSS](https://tailwindcss.com) — A utility-first CSS framework
- [Alpine.js](https://alpinejs.dev) — A rugged, minimal tool for composing behavior
- [Pusher](https://pusher.com) — APIs to enable devs to build realtime features

---

<div align="center">
<p>SIDAMEN &copy; 2024 — Sistem Data Manajemen Mahasiswa</p>
</div>
