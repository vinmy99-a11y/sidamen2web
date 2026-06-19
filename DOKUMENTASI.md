# 📚 DOKUMENTASI TEKNIS — SIDAMEN
## Sistem Data Manajemen Mahasiswa

---

## 1. GAMBARAN UMUM SISTEM

SIDAMEN adalah aplikasi web manajemen data mahasiswa berbasis **Laravel 11** yang dirancang dengan arsitektur **MVC (Model-View-Controller)**. Sistem ini menggabungkan fitur CRUD standar dengan algoritma sorting & searching manual, integrasi file massal, notifikasi email realtime via queue, dan activity log realtime berbasis WebSocket.

### Diagram Arsitektur Sistem

```
┌─────────────────────────────────────────────────────────────────┐
│                        BROWSER (Client)                         │
│         Blade Template + Alpine.js + Tailwind CSS               │
│                    Pusher JS / Laravel Echo                     │
└──────────────────┬──────────────────────────────┬──────────────┘
                   │ HTTP Request                  │ WebSocket
                   ▼                               ▼
┌─────────────────────────────┐    ┌───────────────────────────┐
│       Laravel 11 (Server)   │    │       Pusher Cloud        │
│  ┌─────────────────────┐    │    │  Channel: activity-log    │
│  │  Routes / web.php   │    │    │  Event: ActivityLogCreated│
│  └──────────┬──────────┘    │    └───────────────────────────┘
│             ▼               │              ▲
│  ┌─────────────────────┐    │              │ broadcast()
│  │    Controllers      │────┼──────────────┘
│  │  ┌───────────────┐  │    │
│  │  │ FormRequests  │  │    │    ┌──────────────────────────┐
│  │  └───────────────┘  │    │    │     Queue Worker         │
│  └──────────┬──────────┘    │    │  php artisan queue:work  │
│             ▼               │    │         │                │
│  ┌─────────────────────┐    │    │         ▼                │
│  │      Models         │    │    │  ┌─────────────────┐    │
│  │  Mahasiswa          │    │    │  │  SMTP Gmail     │    │
│  │  Komentar           │    │    │  │  MahasiswaNotif │    │
│  │  ActivityLog        │◄───┼────┘  └─────────────────┘    │
│  │  ImportLog          │    │                               │
│  └──────────┬──────────┘    └───────────────────────────────┘
│             ▼               
│  ┌─────────────────────┐    
│  │   MySQL Database    │    
│  │  mahasiswa          │    
│  │  komentars          │    
│  │  activity_logs      │    
│  │  import_logs        │    
│  │  jobs (queue)       │    
│  │  sessions           │    
│  └─────────────────────┘    
└─────────────────────────────┘
```

---

## 2. STRUKTUR DATABASE

### 2.1 Tabel `mahasiswa`

```sql
CREATE TABLE mahasiswa (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    nim         VARCHAR(12) NOT NULL UNIQUE,
    nama        VARCHAR(100) NOT NULL,
    email       VARCHAR(100) NOT NULL UNIQUE,
    jurusan     VARCHAR(50) NOT NULL,
    angkatan    YEAR NOT NULL,
    created_at  TIMESTAMP NULL,
    updated_at  TIMESTAMP NULL,
    INDEX idx_nim_nama (nim, nama),
    INDEX idx_jurusan (jurusan)
);
```

**Constraint validasi:**
- `nim` — hanya angka, 1–12 digit, unik
- `email` — format valid, unik
- `jurusan` — harus salah satu dari 5 pilihan tetap
- `angkatan` — tahun 2000–sekarang

### 2.2 Tabel `komentars`

```sql
CREATE TABLE komentars (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    mahasiswa_id    BIGINT UNSIGNED NOT NULL,
    user_id         BIGINT UNSIGNED NOT NULL,
    komentar        TEXT NOT NULL,
    created_at      TIMESTAMP NULL,
    updated_at      TIMESTAMP NULL,
    FOREIGN KEY (mahasiswa_id) REFERENCES mahasiswa(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

### 2.3 Tabel `activity_logs`

```sql
CREATE TABLE activity_logs (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    action          VARCHAR(20) NOT NULL,       -- create|update|delete|import|export|login
    description     TEXT NOT NULL,
    subject_type    VARCHAR(50) NULL,           -- 'Mahasiswa'
    subject_id      BIGINT UNSIGNED NULL,
    user_id         BIGINT UNSIGNED NULL,
    causer_name     VARCHAR(100) NULL,
    ip_address      VARCHAR(45) NULL,
    properties      JSON NULL,                  -- {before:{...}, after:{...}}
    created_at      TIMESTAMP NULL,
    updated_at      TIMESTAMP NULL,
    INDEX idx_action_created (action, created_at),
    INDEX idx_user (user_id)
);
```

### 2.4 Tabel `import_logs`

```sql
CREATE TABLE import_logs (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    filename        VARCHAR(255) NOT NULL,
    format          VARCHAR(10) NOT NULL,       -- csv|json
    success_count   INT DEFAULT 0,
    fail_count      INT DEFAULT 0,
    status          VARCHAR(20) DEFAULT 'success', -- success|partial|failed
    errors          JSON NULL,
    user_id         BIGINT UNSIGNED NULL,
    created_at      TIMESTAMP NULL,
    updated_at      TIMESTAMP NULL
);
```

---

## 3. DOKUMENTASI CONTROLLER

### 3.1 AuthController

**File:** `app/Http/Controllers/AuthController.php`

| Method | Route | Fungsi |
|---|---|---|
| `showLogin()` | GET `/login` | Tampilkan halaman login |
| `login(Request)` | POST `/login` | Proses autentikasi + catat log |
| `logout(Request)` | POST `/logout` | Hapus session + catat log |
| `dashboard()` | GET `/dashboard` | Render dashboard dengan statistik |

**Alur `login()`:**
```
1. Validasi input (email, password)
2. try { Auth::attempt() }
3. Jika gagal → back()->with('error', ...)
4. Jika berhasil → session()->regenerate()
5. ActivityLogService::log('login', ...)
6. redirect('/dashboard')
catch { Log::error() → back()->with('error', 'Kesalahan sistem') }
```

**Data untuk dashboard:**
```php
$totalMahasiswa    = Mahasiswa::count();
$totalJurusan      = Mahasiswa::distinct('jurusan')->count();
$importBulanIni    = ImportLog::whereMonth('created_at', now()->month)->count();
$aksiHariIni       = ActivityLog::whereDate('created_at', today())->count();
$distribusiJurusan = Mahasiswa::selectRaw('jurusan, count(*) as total')->groupBy('jurusan')->get();
$recentActivity    = ActivityLog::latest()->limit(5)->get();
```

---

### 3.2 MahasiswaController

**File:** `app/Http/Controllers/MahasiswaController.php`

| Method | Route | Fungsi |
|---|---|---|
| `index()` | GET `/mahasiswa` | Tabel mahasiswa (50/halaman) |
| `create()` | GET `/mahasiswa/create` | Form tambah |
| `store(StoreMahasiswaRequest)` | POST `/mahasiswa` | Simpan + email + log |
| `edit($id)` | GET `/mahasiswa/{id}/edit` | Form edit |
| `update(UpdateMahasiswaRequest, $id)` | PUT `/mahasiswa/{id}` | Update + email + log |
| `destroy($id)` | DELETE `/mahasiswa/{id}` | Hapus + log |
| `getKomentar($id)` | GET `/mahasiswa/{id}/komentar` | JSON list komentar |
| `storeKomentar(Request, $id)` | POST `/mahasiswa/{id}/komentar` | Simpan komentar JSON |

**Alur `store()`:**
```
1. FormRequest otomatis validasi → jika gagal, auto-redirect + errors
2. try {
     Mahasiswa::create($request->validated())
     ActivityLogService::log('create', ...)
     Mail::to($mahasiswa->email)->queue(new MahasiswaNotifikasi($mahasiswa, 'create'))
     redirect()->with('success', ...)
   }
   catch { Log::error() → back()->with('error', ...) }
```

**Response `getKomentar()`:**
```json
[
  {
    "id": 1,
    "komentar": "Teks komentar",
    "user": "Administrator",
    "created_at": "17/06/2024 14:30"
  }
]
```

---

### 3.3 ImportExportController

**File:** `app/Http/Controllers/ImportExportController.php`

| Method | Route | Fungsi |
|---|---|---|
| `index()` | GET `/mahasiswa/import-export` | Halaman import/export |
| `import(Request)` | POST `/mahasiswa/import` | Proses import file |
| `export(Request)` | GET `/mahasiswa/export` | Download CSV/JSON |

**Alur `import()`:**
```
1. Validasi file (mimes: csv,json,txt | max: 5MB)
2. Baca isi file → parse sesuai format
   - JSON: json_decode() → validasi array
   - CSV: explode("\n") → parse header → cek kolom wajib
3. DB::beginTransaction()
4. Loop per baris:
   - Validator::make($row, rules)
   - Jika valid → Mahasiswa::create()
   - Jika tidak → $failCount++, catat error
5. DB::commit()
6. ImportLog::create(summary)
7. ActivityLogService::log('import', ...)
8. redirect()->with('success', ...)
```

**Format CSV yang Valid:**
```
nim,nama,email,jurusan,angkatan
202301001,Budi Santoso,budi@mail.com,Teknik Informatika,2023
```

**Format JSON yang Valid:**
```json
[
  {
    "nim": "202301001",
    "nama": "Budi Santoso",
    "email": "budi@mail.com",
    "jurusan": "Teknik Informatika",
    "angkatan": "2023"
  }
]
```

---

### 3.4 ActivityLogController

**File:** `app/Http/Controllers/ActivityLogController.php`

| Method | Route | Fungsi |
|---|---|---|
| `index()` | GET `/activity-log` | Tampilkan log (50/halaman) |

Data yang dikembalikan ke view sudah di-transform via `->through()` dengan tambahan field `created_at_formatted` dan `time_ago`.

---

## 4. DOKUMENTASI MODEL

### 4.1 Model `Mahasiswa`

```php
// Fillable fields
protected $fillable = ['nim', 'nama', 'email', 'jurusan', 'angkatan'];

// Relationships
public function komentars()  // hasMany(Komentar)

// Scopes
public function scopeByJurusan($query, $jurusan)
public function scopeByAngkatan($query, $angkatan)
public function scopeSearch($query, $keyword)  // LIKE %keyword% on nim, nama, email
```

### 4.2 Model `ActivityLog`

```php
// Fillable
protected $fillable = ['action','description','subject_type','subject_id',
                        'user_id','causer_name','ip_address','properties'];

// Cast
protected $casts = ['properties' => 'array'];

// Appends (auto-append ke JSON)
protected $appends = ['created_at_formatted', 'time_ago'];
```

---

## 5. DOKUMENTASI SERVICE

### 5.1 ActivityLogService

**File:** `app/Services/ActivityLogService.php`

```php
ActivityLogService::log(
    action:      'create',           // string — tipe aksi
    description: 'Deskripsi aksi',  // string — teks log
    subjectType: 'Mahasiswa',        // string|null — nama model
    subjectId:   $id,                // int|null — ID record
    properties:  ['before'=>[], 'after'=>[]] // array|null — data tambahan
);
```

**Cara kerja internal:**
```
1. Auth::user() → ambil user yang login
2. ActivityLog::create([...data...])
3. broadcast(new ActivityLogCreated($log))->toOthers()
   → Kirim ke Pusher channel 'activity-log'
   → Frontend Alpine.js menangkap event dan prepend baris baru ke tabel
4. Jika broadcast gagal → Log::warning() (tidak menghentikan proses)
```

---

## 6. DOKUMENTASI EVENT & BROADCAST

### 6.1 ActivityLogCreated Event

**File:** `app/Events/ActivityLogCreated.php`

```php
// Channel yang digunakan
public function broadcastOn(): array
{
    return [new Channel('activity-log')]; // Public channel
}

// Nama event di frontend
public function broadcastAs(): string
{
    return 'ActivityLogCreated';
}
```

**Payload yang dikirim ke frontend:**
```json
{
  "id": 42,
  "action": "create",
  "description": "Mahasiswa baru ditambahkan: Budi (202301001)",
  "subject_type": "Mahasiswa",
  "subject_id": 15,
  "causer_name": "Administrator",
  "ip_address": "127.0.0.1",
  "created_at_formatted": "17/06/2024 14:30:00",
  "time_ago": "Baru saja"
}
```

**Cara frontend mendengarkan:**
```javascript
// Di activity-log/index.blade.php
const pusher = new Pusher('PUSHER_KEY', { cluster: 'ap1', encrypted: true });
pusher.subscribe('activity-log')
      .bind('ActivityLogCreated', (data) => {
          // Prepend data ke tabel tanpa refresh
          this.allLogs.unshift({...data, _new: true});
      });
```

---

## 7. DOKUMENTASI VALIDASI

### 7.1 StoreMahasiswaRequest

| Field | Rules | Pesan Error |
|---|---|---|
| `nim` | required, digits_between:1,12, unique:mahasiswa | NIM harus angka maks 12 digit / sudah terdaftar |
| `nama` | required, string, min:3, max:100 | Nama minimal 3 karakter |
| `email` | required, email:rfc, unique:mahasiswa, regex | Format email tidak valid / sudah digunakan |
| `jurusan` | required, in:[5 pilihan] | Jurusan tidak tersedia |
| `angkatan` | required, integer, min:2000, max:tahunini | Angkatan tidak valid |

**Regex email yang digunakan:**
```
/^[^\s@]+@[^\s@]+\.[a-zA-Z]{2,}$/
```

### 7.2 Validasi Frontend (Alpine.js)

Validasi dijalankan **sebelum form di-submit** via event `@submit="validateAll"`:

```javascript
validateNim() {
    if (!/^\d+$/.test(v))   → error: 'NIM hanya boleh berisi angka'
    if (v.length > 12)      → error: 'NIM maksimal 12 digit'
}
validateEmail() {
    const re = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!re.test(email))    → error: 'Format email tidak valid'
}
```

> Double validation: frontend untuk UX, backend untuk keamanan.

---

## 8. DOKUMENTASI ALGORITMA

### 8.1 Linear Search

**Kompleksitas:** `O(n)` waktu, `O(1)` ruang

```javascript
for (let i = 0; i < arr.length; i++) {
    const match = arr[i].nim.toLowerCase().includes(q)
               || arr[i].nama.toLowerCase().includes(q);
    buildLog.push({ nim: arr[i].nim, nama: arr[i].nama, found: match });
    if (match && result === null) result = arr[i];
}
```

- Mendukung pencarian **parsial** (contains)
- Menampilkan **semua** baris yang cocok
- Build log mencatat **semua** baris yang diperiksa

### 8.2 Binary Search

**Kompleksitas:** `O(log n)` waktu, `O(1)` ruang
**Syarat:** Array harus diurutkan terlebih dahulu

```javascript
const sorted = [...arr].sort((a, b) => a.nim.localeCompare(b.nim));
let lo = 0, hi = sorted.length - 1;
while (lo <= hi) {
    const mid = Math.floor((lo + hi) / 2);
    buildLog.push({ nim: sorted[mid].nim, nama: sorted[mid].nama, found: match });
    if (match)       { result = sorted[mid]; break; }
    else if (val < q) lo = mid + 1;
    else              hi = mid - 1;
}
```

- Hanya mendukung pencarian **exact match** NIM
- Jauh lebih cepat untuk dataset besar
- Build log hanya mencatat baris yang **benar-benar diperiksa** (log n baris)

### 8.3 Bubble Sort

**Kompleksitas:** `O(n²)` waktu, `O(1)` ruang

```javascript
for (let i = 0; i < arr.length - 1; i++)
    for (let j = 0; j < arr.length - 1 - i; j++)
        if (arr[j][key].localeCompare(arr[j+1][key]) > 0)
            [arr[j], arr[j+1]] = [arr[j+1], arr[j]];
```

### 8.4 Selection Sort

**Kompleksitas:** `O(n²)` waktu, `O(1)` ruang

```javascript
for (let i = 0; i < arr.length - 1; i++) {
    let minIdx = i;
    for (let j = i+1; j < arr.length; j++)
        if (arr[j][key] < arr[minIdx][key]) minIdx = j;
    if (minIdx !== i) [arr[i], arr[minIdx]] = [arr[minIdx], arr[i]];
}
```

### 8.5 Shell Sort

**Kompleksitas:** `O(n log n)` rata-rata, `O(1)` ruang

```javascript
let gap = Math.floor(arr.length / 2);
while (gap > 0) {
    for (let i = gap; i < arr.length; i++) {
        const temp = arr[i];
        let j = i;
        while (j >= gap && arr[j-gap][key] > temp[key]) {
            arr[j] = arr[j-gap]; j -= gap;
        }
        arr[j] = temp;
    }
    gap = Math.floor(gap / 2);
}
```

---

## 9. DOKUMENTASI EMAIL

### 9.1 MahasiswaNotifikasi Mailable

**File:** `app/Mail/MahasiswaNotifikasi.php`

```php
// Implementasi ShouldQueue → masuk ke jobs table
class MahasiswaNotifikasi extends Mailable implements ShouldQueue

// Trigger: create
Mail::to($mahasiswa->email)->queue(new MahasiswaNotifikasi($mahasiswa, 'create'));

// Trigger: update
Mail::to($mahasiswa->email)->queue(new MahasiswaNotifikasi($mahasiswa, 'update'));
```

**Subject email:**
- Create: `✅ Selamat Datang di SIDAMEN — Data Anda Telah Terdaftar`
- Update: `📝 Notifikasi Perubahan Data — SIDAMEN`

**Template:** `resources/views/emails/mahasiswa-notifikasi.blade.php`

### 9.2 Alur Queue

```
Controller → Mail::queue() → jobs table (DB)
                                    ↓
                          php artisan queue:work
                                    ↓
                           smtp.gmail.com:587
                                    ↓
                           Inbox mahasiswa
```

---

## 10. KONFIGURASI LENGKAP

### 10.1 File `.env` Lengkap

```env
APP_NAME=SIDAMEN
APP_ENV=local
APP_KEY=base64:xxxxxxxxxxxxx  ← otomatis saat php artisan key:generate
APP_DEBUG=true
APP_URL=http://localhost:8000

LOG_CHANNEL=stack
LOG_LEVEL=debug

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=sidamen
DB_USERNAME=root
DB_PASSWORD=

QUEUE_CONNECTION=database

BROADCAST_DRIVER=pusher
PUSHER_APP_ID=your-app-id
PUSHER_APP_KEY=your-app-key
PUSHER_APP_SECRET=your-app-secret
PUSHER_APP_CLUSTER=ap1

MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=your@gmail.com
MAIL_PASSWORD=your-app-password-16char
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=your@gmail.com
MAIL_FROM_NAME="SIDAMEN"

SESSION_DRIVER=database
SESSION_LIFETIME=120
```

### 10.2 Perintah Artisan Lengkap

```bash
# Setup awal (jalankan sekali)
php artisan key:generate
php artisan migrate
php artisan queue:table && php artisan session:table && php artisan migrate
php artisan db:seed --class=AdminSeeder

# Run development (jalankan di terminal terpisah)
php artisan serve          # Terminal 1 — web server
php artisan queue:work     # Terminal 2 — email queue

# Utility
php artisan route:list     # Lihat semua route
php artisan migrate:fresh  # Reset database (HAPUS SEMUA DATA)
php artisan queue:flush    # Hapus semua job dari queue
php artisan cache:clear    # Bersihkan cache
```

---

## 11. TROUBLESHOOTING

| Error | Penyebab | Solusi |
|---|---|---|
| `Class 'Pusher\Pusher' not found` | Package belum diinstall | `composer require pusher/pusher-php-server` |
| `Target class [EventServiceProvider] does not exist` | Provider belum didaftarkan | Tambahkan ke `bootstrap/providers.php` |
| Email tidak terkirim | Queue worker tidak jalan | Jalankan `php artisan queue:work` |
| Email tidak terkirim | App Password salah | Generate ulang di Google Account |
| Pusher tidak connect | Key salah / cluster salah | Cek kembali `.env` Pusher credentials |
| `SQLSTATE: Table doesn't exist` | Migration belum dijalankan | `php artisan migrate` |
| `419 Page Expired` | CSRF token expired | Refresh halaman |
| Import gagal semua | Format kolom salah | Pastikan header CSV: `nim,nama,email,jurusan,angkatan` |

---

*Dokumentasi ini berlaku untuk SIDAMEN versi 1.0.0*
