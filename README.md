# XML → IP CSV Converter (Browser-side)

Aplikasi web single-page yang berjalan 100% di browser (client-side) untuk:
- Membaca file XML berisi tag <definition> yang di-encode Base64 dari hasil PHP serialize.
- Melakukan decode Base64 → unserialize PHP → ambil field "definedIPs".
- Parsing daftar IP (single IP, range, CIDR).
- Ekspansi menjadi IP per-satuan.
- Export hasil ke CSV (1 IP per baris).

Tidak ada backend / server-side processing. Semua logika di JavaScript murni.

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

server {
    listen 80;
    server_name ip-converter.local;

    root /var/www/ip-converter;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}

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

Langkah:

1. Simpan sebagai `/etc/apache2/sites-available/ip-converter.conf`
2. Enable site dan reload:

   a2ensite ip-converter.conf
   apachectl configtest
   systemctl reload apache2

3. Tambah di `/etc/hosts`:

   127.0.0.1   ip-converter.local

4. Akses:

   http://ip-converter.local

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
