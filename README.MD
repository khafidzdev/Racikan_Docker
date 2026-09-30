# Panduan Deployment dan Instalasi Docker Laravel (FrankenPHP)

Dokumentasi instalasi, konfigurasi, dan deployment aplikasi Laravel menggunakan Docker Compose dan FrankenPHP (PHP 8.4 Alpine), lengkap dengan setup domain via Cloudflare Tunnel atau Reverse Proxy Nginx.

---

## Prasyarat Sistem

Pastikan server telah terpasang:
- Git
- Docker Engine dan Docker Compose v2 (`docker compose`)
- Nginx (jika menggunakan metode Reverse Proxy lokal)
- Cloudflare Tunnel / `cloudflared` (jika menggunakan metode Cloudflare Tunnel)

---

## Langkah Instalasi

### 1. Clone Repositori

```bash
cd /var/www
git clone <URL_REPOSITORY_ANDA> <nama_folder_project>
cd <nama_folder_project>
```

---

### 2. Composer Install

Jalankan instalasi dependensi vendor:

```bash
composer install --optimize-autoloader --no-dev
```

---

### 3. Setting Environment (.env)

Salin file konfigurasi dan sesuaikan isinya:

```bash
cp .env.example .env
nano .env
```

Pastikan variabel database dan environment disesuaikan:

```dotenv
APP_NAME=LaravelApp
APP_ENV=production
APP_DEBUG=false
APP_URL=[https://domain-anda.com](https://domain-anda.com)

# Konfigurasi Database
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=nama_db
DB_USERNAME=root
DB_PASSWORD=rahasia
```

---

### 4. Build Docker Image

Build image FrankenPHP tanpa menggunakan cache:

```bash
docker compose build --no-cache
```

---

### 5. Menjalankan Container

Jalankan container di latar belakang (detached mode):

```bash
docker compose up -d
```

Pastikan container sudah aktif:

```bash
docker compose ps
```

---

### 6. Setup Awal Laravel di Dalam Container

Eksekusi perintah inisialisasi framework berikut langsung di dalam container:

```bash
docker compose exec app php artisan key:generate
docker compose exec app php artisan storage:link
docker compose exec app php artisan migrate --force
docker compose exec app php artisan optimize
```

---

## Menghubungkan Domain ke Container

Pilih salah satu metode di bawah ini sesuai arsitektur jaringan yang digunakan:

### Opsi A: Menggunakan Cloudflare Tunnel

Jika service `cloudflared` sudah terpasang dan aktif di server host:

1. Buka **Cloudflare Zero Trust** -> **Networks** -> **Tunnels**.
2. Pilih Tunnel server yang aktif -> klik **Configure**.
3. Masuk ke tab **Public Hostname** -> klik **Add a public hostname**.
4. Isi data routing:
   - **Subdomain / Domain:** Tentukan domain atau subdomain (contoh: `app` dan `domainanda.com`).
   - **Type:** `HTTP`
   - **URL:** `localhost:8000` (atau port host yang diatur pada `docker-compose.yml`).
5. Klik **Save hostname**.

---

### Opsi B: Menggunakan Nginx Reverse Proxy di Host Server

Jika menggunakan web server Nginx bawaan server host untuk mengarahkan trafik domain ke container:

#### 1. Buat Konfigurasi Virtual Host

Buat file konfigurasi baru di direktori `sites-available`:

```bash
nano /etc/nginx/sites-available/<nama_domain>.conf
```

Isi dengan konfigurasi reverse proxy berikut:

```nginx
server {
    listen 80;
    server_name domain-anda.com [www.domain-anda.com](https://www.domain-anda.com);

    # Meneruskan IP asli client (termasuk jika di-proxy Cloudflare)
    set_real_ip_from 173.245.48.0/20;
    set_real_ip_from 103.21.244.0/22;
    set_real_ip_from 103.22.200.0/22;
    set_real_ip_from 103.31.4.0/22;
    set_real_ip_from 141.101.64.0/18;
    set_real_ip_from 108.162.192.0/18;
    set_real_ip_from 190.93.240.0/20;
    set_real_ip_from 188.114.96.0/20;
    set_real_ip_from 197.234.240.0/22;
    set_real_ip_from 198.41.128.0/17;
    set_real_ip_from 162.158.0.0/15;
    set_real_ip_from 104.16.0.0/13;
    set_real_ip_from 104.24.0.0/14;
    set_real_ip_from 172.64.0.0/13;
    set_real_ip_from 131.0.72.0/22;
    real_ip_header CF-Connecting-IP;

    client_max_body_size 100M;

    location / {
        proxy_pass [http://127.0.0.1:8000](http://127.0.0.1:8000);
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### 2. Aktifkan Site dan Restart Nginx

```bash
ln -s /etc/nginx/sites-available/<nama_domain>.conf /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

#### 3. Pasang SSL Gratis (Certbot) jika tidak memakai Flexible SSL Cloudflare

```bash
certbot --nginx -d domain-anda.com -d [www.domain-anda.com](https://www.domain-anda.com)
```

---

## Penyesuaian Force HTTPS di Laravel

Agar seluruh file CSS, JS, dan route Laravel otomatis menggunakan skema `https://` tanpa memicu isu *mixed content*, pastikan file `app/Providers/AppServiceProvider.php` memiliki konfigurasi berikut:

```php
use Illuminate\Support\Facades\URL;

public function boot(): void
{
    if (config('app.env') === 'production') {
        URL::forceScheme('https');
    }
}
```

Setelah memperbarui provider, bersihkan dan perbarui cache:

```bash
docker compose exec app php artisan optimize:clear
docker compose exec app php artisan optimize
```

---

## Perintah Operasional dan Maintenance

### Cek Log Container (Real-Time)
```bash
docker compose logs -f --tail=50 app
```

### Masuk ke Shell Container
```bash
docker compose exec app sh
```

### Restart Container
```bash
docker compose restart app
```

### Refresh Cache Setelah Perubahan Code atau .env
```bash
docker compose exec app php artisan optimize:clear
docker compose exec app php artisan optimize
```

### Matikan Container
```bash
docker compose down
```

---

## Verifikasi Akses

Lakukan pengujian langsung di server:

```bash
curl -I [http://127.0.0.1:8000](http://127.0.0.1:8000)
```
