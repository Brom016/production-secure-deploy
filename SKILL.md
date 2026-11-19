---
name: production-secure-deploy
description: Aturan wajib saat membuat, mereview, atau men-deploy kode untuk production agar aman dan sesuai standar industri (OWASP Top 10, ASVS, 12-factor). Gunakan untuk aplikasi web/API (PHP, Laravel, Node, dll), konfigurasi server, upload file, autentikasi, database, dan deploy ke hosting. Gunakan juga jika user menyebut cPanel, shared hosting, File Manager, atau tidak punya akses terminal.
---

# Production Secure Deploy

Standar acuan: OWASP Top 10, OWASP ASVS level 1 sebagai minimum, 12-factor app untuk konfigurasi. Aturan di bawah bersifat wajib kecuali user menyatakan secara eksplisit bahwa itu hanya prototype lokal.

## Aturan kerja dan efisiensi

1. Baca hanya bagian yang relevan dengan tugas. Muat `references/cpanel-no-terminal.md` hanya jika deploy ke cPanel/shared hosting/tanpa terminal.
2. Tulis kode final langsung. Jangan menjelaskan konsep dasar kecuali ditanya.
3. Jika stack, versi PHP/Node, atau jenis data (data pribadi, pembayaran, kesehatan) belum diketahui dan mengubah rekomendasi, tanyakan satu kali di awal. Jika tidak mengubah, tulis asumsi dalam satu baris lalu lanjut.
4. Saat mengubah kode yang sudah ada, kirim hanya bagian yang berubah, bukan seluruh file.
5. Akhiri dengan maksimal 5 butir risiko tersisa yang spesifik pada tugas itu. Jangan menulis checklist generik yang tidak relevan.
6. Jangan menyatakan sesuatu "sudah aman" atau "aman 100%". Katakan apa yang sudah ditutup dan apa yang belum diverifikasi.
7. Jika permintaan user melanggar aturan di bawah (misalnya `APP_DEBUG=true` di production, chmod 777, hardcode password), tolak bagian itu, jelaskan risikonya dalam satu atau dua kalimat, dan berikan alternatif yang berfungsi.

## Secrets dan konfigurasi

- Jangan pernah hardcode secret (password DB, API key, APP_KEY, token) di kode, HTML, JS frontend, atau repo. Gunakan environment/`.env`.
- File `.env` dan config sensitif ditaruh di luar web root. Jika terpaksa di dalam web root, blokir akses lewat server config.
- `.env.example` hanya berisi nama variabel dengan nilai kosong atau palsu.
- Kredensial berbeda untuk dev, staging, production. Jangan pakai ulang.
- Jika secret pernah ter-commit atau ter-upload publik, anggap bocor dan rotasi, bukan sekadar hapus.
- Mode debug dan stack trace harus mati di production. Pesan error ke user bersifat generik, detail masuk log.

## Input, output, dan database

- Semua query memakai prepared statement/parameter binding atau ORM. Dilarang string concatenation ke SQL, termasuk untuk `ORDER BY` dan nama kolom (gunakan allowlist).
- Validasi input di server (tipe, panjang, format, allowlist). Validasi di frontend hanya untuk UX.
- Escape output sesuai konteks (HTML, atribut, JS, URL). Di Blade gunakan `{{ }}`, bukan `{!! !!}` untuk data user. Di React hindari `dangerouslySetInnerHTML`.
- Mass assignment: definisikan `$fillable` eksplisit, jangan `$guarded = []`.
- User database aplikasi hanya punya hak yang dibutuhkan (SELECT/INSERT/UPDATE/DELETE pada database itu saja), bukan root atau ALL PRIVILEGES global.
- Redirect hanya ke URL internal atau allowlist (hindari open redirect). Request ke URL dari user harus dibatasi (hindari SSRF).

## Autentikasi dan otorisasi

- Password di-hash dengan Argon2id atau bcrypt. Dilarang MD5, SHA1, SHA256 polos, atau enkripsi reversible.
- Otorisasi dicek di server pada setiap request ke resource, termasuk kepemilikan data (cegah IDOR). Menyembunyikan tombol di UI bukan otorisasi.
- Rate limit pada login, reset password, OTP, dan endpoint mahal. Pertimbangkan lockout bertahap.
- Session: regenerate ID setelah login, cookie dengan `Secure`, `HttpOnly`, `SameSite=Lax` atau `Strict`, ada timeout.
- Proteksi CSRF aktif untuk semua request yang mengubah state. API token disimpan hash-nya, bisa dicabut, dan punya scope.
- Pesan login dan reset password tidak membocorkan apakah akun ada.

## Upload file

- Validasi ekstensi dan MIME dengan allowlist, batasi ukuran, ganti nama file dengan nama acak buatan server.
- Simpan di luar web root atau pastikan eksekusi script dimatikan di folder upload. Jangan pernah izinkan `.php`, `.phtml`, `.phar`, `.htaccess` terupload.
- Gambar diproses ulang (re-encode) bila memungkinkan. Download file privat lewat controller yang mengecek otorisasi.

## Transport dan header

- HTTPS wajib, redirect HTTP ke HTTPS. Aktifkan HSTS setelah HTTPS terverifikasi stabil.
- Header minimum: `X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN` (atau CSP `frame-ancestors`), `Referrer-Policy`, `Permissions-Policy`. CSP disusun sesuai aplikasi dan diuji, jangan menyalin CSP generik yang merusak halaman atau terlalu longgar (`unsafe-inline` + `*`).
- CORS: sebutkan origin eksplisit. Dilarang `*` bersama credentials.
- Matikan directory listing. Blokir akses ke dotfile (`.env`, `.git`), file backup (`.sql`, `.bak`, `.zip`), dan `composer.json/lock`, `package.json` dari web.

## Dependensi dan build

- Production hanya berisi dependensi production (`composer install --no-dev --optimize-autoloader`, `npm ci` lalu build).
- Kunci versi lewat lockfile. Jalankan `composer audit` / `npm audit` sebelum rilis, dan catat temuan yang diabaikan beserta alasannya.
- Jangan memuat script dari CDN tanpa kebutuhan jelas. Jika perlu, pin versi dan gunakan SRI.
- Jangan sertakan `.git`, `tests`, `node_modules`, file dokumentasi internal, atau dump database di paket yang diupload.

## Logging, backup, dan operasional

- Log kejadian keamanan (login gagal, perubahan hak akses, error 5xx). Jangan mencatat password, token, nomor kartu, atau data pribadi penuh.
- Backup database dan file terjadwal, simpan di luar server yang sama, dan uji restore minimal sekali. Backup yang belum pernah diuji dianggap tidak ada.
- Siapkan rencana rollback sebelum deploy: salinan versi sebelumnya tetap ada.
- Terapkan prinsip least privilege pada akun hosting, FTP, dan database. Aktifkan 2FA pada akun hosting dan registrar domain.
- Data pribadi: kumpulkan seminimal mungkin dan sebutkan jika ada implikasi UU PDP (Indonesia) pada data yang diproses.

## Gerbang sebelum deploy

Jangan menyatakan siap deploy sebelum semua ini terjawab:

1. Debug mati, secret tidak ada di kode atau paket upload.
2. Semua endpoint yang mengubah data punya autentikasi, otorisasi, dan CSRF.
3. Tidak ada query atau output yang rentan injeksi.
4. HTTPS aktif dan file sensitif tidak bisa diakses dari URL.
5. Backup dan rollback tersedia.

## Batas yang harus diakui

Skill ini mengurangi kesalahan umum, tetapi tidak menggantikan code review manusia, pengujian keamanan, atau audit independen. Untuk aplikasi dengan pembayaran, data kesehatan, atau data pribadi volume besar, sebutkan bahwa shared hosting memiliki keterbatasan isolasi dan kontrol, lalu sarankan VPS atau platform terkelola.
