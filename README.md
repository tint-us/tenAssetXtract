# TenAssetXtract

## Background Story

## Latar Belakang

1. Saya menggunakan Tenable Security Center (Tenable SC) dan perlu melakukan export seluruh Asset yang berbasis metode **Static IP List**.
2. Hasil export yang diperoleh ternyata bukan berupa daftar IP Address, melainkan sebuah file **XML** yang di dalamnya menyimpan data dalam bentuk **Base64 dari PHP-serialized object**.
3. Untuk keperluan analisis dan pemrosesan lanjutan, saya membutuhkan **daftar IP Address** yang dapat digunakan secara langsung (plaintext, satu IP per baris).
4. Berdasarkan kebutuhan tersebut, saya membuat aplikasi sederhana ini untuk membantu proses ekstraksi IP, serta mempermudah pengguna Tenable lain yang menghadapi permasalahan serupa.

Dari problem kecil tapi repeatable inilah **TenAssetXtract** lahir.

### Inti Masalah yang Diselesaikan

- Aplikasi ini digunakan untuk melakukan **extract XML** yang berisikan asset dari hasil export di **Tenable SC** dengan type asset **"Static IP List"**
- Pada Tenable SC, menu yang digunakan adalah **Assets - Assets** - Centang pada “Static IP List” yang dimaksud. Klik **export** pada atas panel

![Alt Text](https://github.com/user-attachments/assets/790fd6bf-77d7-4960-aa2d-554ab15a05b4)

### Solution

- Data IP tersimpan di dalam **PHP-serialized object** yang telah di-encode **Base64**
- Untuk membaca field `definedIPs`, dibutuhkan proses decode & parsing manual
- Efisiensi turun, risiko human error naik, dan waktunya kebuang di hal yang seharusnya bisa diotomasi

**TenAssetXtract** melakukan proses:

1. Membaca XML hasil export Tenable Security Center
2. Decode isi Base64 di dalamnya
3. Unserialize subset PHP-object
4. Ambil field `definedIPs`
5. Ekspansi IP (single, range, CIDR) menjadi 1 IP per baris
6. Deduplikasi & sorting
7. Tampilkan dan export kembali menjadi file **IP list yang usable**

### Audience / Pengguna

Tool ini ditujukan untuk:

- Pengguna Tenable Security Center (Tenable SC)
- Yang pernah export Static IP List
- Dan ternyata output-nya adalah XML berisi Base64 dari PHP-serialized object
- Butuh **IP list usable** untuk diproses lebih lanjut

### Kenapa Tool Ini Dibuat di Browser

Karena saya ingin tool-nya:

- **Ringan, cepat, portable**
- **No server hosting required**
- Bisa dibuka langsung via browser atau diserve lewat container `"nginx:alpine"` jika dibutuhkan

---

> Tidak semua tool harus besar untuk menyelesaikan problem besar. Terkadang yang kita butuhkan cuma menyelesaikan problem kecil yang selalu muncul dan mengganggu flow kerja kita.

Kalau Anda membaca ini dan merasa, *“Wah ini gue banget”*, berarti kita pernah mengalami problem yang sama. Semoga TenAssetXtract bisa menghemat waktu Anda seperti waktu saya dulu diselamatkan tool ini.

Happy Xtracting ✌️

---

## Fitur Utama

- Input XML via:
  - Upload file (.xml)
  - Drag & drop file
  - Paste langsung isi XML ke textarea
- Parsing:
  - XML → tag <definition>
  - Base64 decode (atob)
  - PHP unserialize (subset tipe: a, s, i, N)
  - Ambil field "definedIPs"
- Format supported di "definedIPs":
  - IP tunggal: 10.216.0.18
  - Range: 10.216.0.112-10.216.0.114
  - CIDR: 10.216.130.30/31, 10.216.130.56/30
- Ekspansi:
  - Range → semua IP dari X sampai Y
  - CIDR → semua alamat di subnet tersebut
- Dedup dan sort:
  - Remove duplikat dengan Set
  - Sorting IP dengan konversi ip→int32
- Output:
  - Tabel list IP di halaman
  - Tombol Download CSV (header: "IP Address")

---

## Guard / Batasan

Untuk mencegah browser freeze, ada guard:

- MAX_EXPANDED_IPS = 1.000.000
- Sebelum ekspansi, semua item di-estimate:
  - Range: hitung (end - start + 1)
  - CIDR: 2^(32 - prefix)
- Kalau total estimasi > limit → aplikasi akan throw error
  dan menginstruksikan user untuk cek konfigurasi range / CIDR.

---

## Cara Pakai (Lokal, Tanpa Web Server)

1. Simpan file `index.html` (dari proyek ini) di satu folder.
2. Buka `index.html` di browser:
   - Double click, atau
   - Dari browser: Ctrl+O → pilih file `index.html`.
3. Di halaman:
   - Drag & drop file XML, atau
   - Klik upload untuk pilih file XML, atau
   - Paste isi XML ke textarea.
4. Klik tombol "Convert to CSV".
5. Kalau berhasil:
   - Tabel IP akan muncul
   - Tombol "Download CSV" aktif → klik untuk mengunduh `export_ips.csv`.

---

## Deployment di Web Server Native

### 1. Nginx

Misal struktur direktori:

- Document root untuk app ini:
  - `/var/www/ip-converter/`
- Isi minimal:
  - `/var/www/ip-converter/index.html`

Contoh konfigurasi server block:
```
server {
    listen 80;
    server_name ip-converter.local;

    root /var/www/ip-converter;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Langkah:

1. Simpan config di file, misalnya:
   - `/etc/nginx/sites-available/ip-converter.conf`
2. Enable site (kalau pakai Debian/Ubuntu):
   - `ln -s /etc/nginx/sites-available/ip-converter.conf /etc/nginx/sites-enabled/ip-converter.conf`
3. Tambah entry di `/etc/hosts`:

   127.0.0.1   ip-converter.local

4. Test config dan reload:

   nginx -t
   systemctl reload nginx

5. Akses di browser:

   http://ip-converter.local

---

### 2. Apache HTTPD

Struktur direktori:

- `/var/www/ip-converter/index.html`

Contoh VirtualHost:
```
<VirtualHost *:80>
    ServerName ip-converter.local
    DocumentRoot /var/www/ip-converter

    <Directory /var/www/ip-converter>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/ip-converter-error.log
    CustomLog ${APACHE_LOG_DIR}/ip-converter-access.log combined
</VirtualHost>
```
Langkah:

1. Simpan sebagai `/etc/apache2/sites-available/ip-converter.conf`
2. Enable site dan reload:
```
   a2ensite ip-converter.conf
   apachectl configtest
   systemctl reload apache2
```
3. Tambah di `/etc/hosts`:
```
   127.0.0.1   ip-converter.local
```
4. Akses:
```
   http://ip-converter.local
```
---

## Deployment via Docker

Aplikasi ini adalah aplikasi statis (HTML + JS). Cara paling simpel adalah menggunakan image `nginx:alpine`.

### Dockerfile
```
FROM nginx:alpine

# hapus default html nginx
RUN rm -f /usr/share/nginx/html/*

# copy aplikasi
COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```
### Build Image

Di folder yang berisi `index.html` dan `Dockerfile`, jalankan:
```
docker build -t ip-xml-converter .
```
### Run Container
```
docker run --rm -p 8080:80 --name ip-xml-converter ip-xml-converter
```
Lalu akses di browser:
```
http://localhost:8080
```
---

## Docker Compose (Opsional)

Buat file `docker-compose.yml`:
```
version: "3.8"

services:
  ip-xml-converter:
    image: ip-xml-converter
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:80"
    restart: unless-stopped
```
Jalankan:
```
docker compose up --build -d
```
Akses:
```
http://localhost:8080
```
---

## Catatan Teknis Implementasi

- Unserialize PHP di JavaScript:
  - Hanya support subset tipe: a (array), s (string), i (integer), N (null).
  - Cukup untuk struktur serialize seperti:
    - a:2:{ s:"definedIPs"; s:"..."; s:"assetDataFields"; a:0:{} }

- IP Handling:
  - Validasi IPv4: 4 oktet, 0–255.
  - ipToInt: konversi ip ke integer 32-bit tanpa signed.
  - intToIp: kebalikan ipToInt.
  - Range:
    - "A.B.C.X-A.B.C.Y" → expand dari X..Y (inklusif).
  - CIDR:
    - hitung jumlah host: 2^(32-prefix)
    - network base: (baseInt & mask)
    - loop 0..hostCount-1 untuk isi semua IP.

---

## Possible Future Extension

- Tambah kolom di CSV:
  - Nama asset / nama file sumber.
- Tambah mode:
  - Export dalam format JSON.
- Integrasi:
  - Endpoint kecil (Python/Node) jika mau proses server-side.
