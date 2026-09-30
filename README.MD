# Panduan Deployment dan Instalasi Docker Laravel (FrankenPHP)

Dokumentasi deployment universal untuk aplikasi Laravel menggunakan Docker Compose dan FrankenPHP (PHP 8.4 Alpine), dilengkapi opsi Reverse Proxy menggunakan Nginx, Apache2, maupun Cloudflare Tunnel.

---

## Prasyarat Sistem

Pastikan server telah terpasang:
- Git
- Docker Engine dan Docker Compose v2 (`docker compose`)
- Salah satu dari: Nginx, Apache2, atau Cloudflare Tunnel

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

Salin file konfigurasi dan sesuaikan nilainya:

```bash
cp .env.example .env
nano .env
```

Sesuaikan variabel penting:

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

Build image FrankenPHP tanpa cache:

```bash
docker compose build --no-cache
```

---

### 5. Menjalankan Container

Jalankan container di latar belakang:

```bash
docker compose up -d
```

Pastikan container berstatus running:

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

Pilih salah satu metode reverse proxy yang sesuai dengan web server di host VPS Anda:

---

### Opsi A: Menggunakan Nginx (Host Server)

#### 1. Buat File Virtual Host Nginx
```bash
nano /etc/nginx/sites-available/<nama_domain>.conf
```

Isi konfigurasi:
```nginx
server {
    listen 80;
    server_name domain-anda.com [www.domain-anda.com](https://www.domain-anda.com);

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

#### 2. Aktifkan dan Reload Nginx
```bash
ln -s /etc/nginx/sites-available/<nama_domain>.conf /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

---

### Opsi B: Menggunakan Apache2 (Host Server)

#### 1. Aktifkan Modul Proxy Apache
```bash
a2enmod proxy proxy_http headers
systemctl restart apache2
```

#### 2. Buat File Virtual Host Apache
```bash
nano /etc/apache2/sites-available/<nama_domain>.conf
```

Isi konfigurasi:
```apache
<VirtualHost *:80>
    ServerName domain-anda.com
    ServerAlias [www.domain-anda.com](https://www.domain-anda.com)

    ProxyPreserveHost On
    ProxyPass / [http://127.0.0.1:8000/](http://127.0.0.1:8000/)
    ProxyPassReverse / [http://127.0.0.1:8000/](http://127.0.0.1:8000/)

    RequestHeader set X-Forwarded-Proto "https"
    RequestHeader set X-Forwarded-Port "443"

    ErrorLog ${APACHE_LOG_DIR}/<nama_domain>_error.log
    CustomLog ${APACHE_LOG_DIR}/<nama_domain>_access.log combined
</VirtualHost>
```

#### 3. Aktifkan dan Reload Apache
```bash
a2ensite <nama_domain>.conf
apache2ctl configtest
systemctl reload apache2
```

---

### Opsi C: Menggunakan Cloudflare Tunnel (Tanpa Nginx / Tanpa Apache)

Jika host sudah memiliki service `cloudflared` yang aktif:

1. Buka **Cloudflare Zero Trust** -> **Networks** -> **Tunnels**.
2. Pilih Tunnel yang aktif -> klik **Configure**.
3. Masuk ke tab **Public Hostname** -> klik **Add a public hostname**.
4. Isi data routing:
   - **Subdomain / Domain:** Tentukan domain atau subdomain target.
   - **Type:** `HTTP`
   - **URL:** `localhost:8000` (atau port host pada `docker-compose.yml`).
5. Klik **Save hostname**.

---

## Penyesuaian Force HTTPS di Laravel

Agar asset CSS, JS, dan seluruh URL Laravel otomatis menggunakan HTTPS saat berada di balik proxy:

Tambahkan kode ini pada file `app/Providers/AppServiceProvider.php`:

```php
use Illuminate\Support\Facades\URL;

public function boot(): void
{
    if (config('app.env') === 'production') {
        URL::forceScheme('https');
    }
}
```

Segarkan cache Laravel di container:

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

### Refresh Cache Setelah Update Code / .env
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

Uji endpoint langsung di server lokal:

```bash
curl -I [http://127.0.0.1:8000](http://127.0.0.1:8000)
```
