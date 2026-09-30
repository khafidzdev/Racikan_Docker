
## Prasyarat Sistem

Pastikan server telah terpasang:
- Git
- Docker Engine dan Docker Compose v2 (`docker compose`)


## Langkah Instalasi

### 1. Clone Repositori

cd /var/www
git clone <URL_REPOSITORY_ANDA> <nama_folder_project>
cd <nama_folder_project>


### 2. Composer Install

Jalankan instalasi dependensi:

composer install --optimize-autoloader --no-dev

### 3. Setting Environment (.env)

Salin file konfigurasi dan sesuaikan isinya:

cp .env.example .env
nano .env

Pastikan variabel database dan environment disesuaikan:

APP_NAME=LaravelApp
APP_ENV=production
APP_DEBUG=false
APP_URL=http://127.0.0.1:8000

# Konfigurasi Database
DB_CONNECTION=mysql
DB_HOST=127.0.0.1 #Pakai ip public jika di vps
DB_PORT=3306
DB_DATABASE=nama_db
DB_USERNAME=root
DB_PASSWORD=


### 4. Build Docker Image

Build image FrankenPHP tanpa menggunakan cache:

docker compose build --no-cache


### 5. Menjalankan Container

Jalankan container di latar belakang (detached mode):

docker compose up -d

Pastikan container sudah aktif:

docker compose ps


### 6. Setup Awal Laravel di Dalam Container

Eksekusi perintah inisialisasi framework berikut langsung di dalam container:

docker compose exec app php artisan key:generate

docker compose exec app php artisan storage:link

docker compose exec app php artisan migrate --force

docker compose exec app php artisan optimize


## Perintah Operasional dan Maintenance

### Cek Log Container (Real-Time)
docker compose logs -f --tail=50 app

### Masuk ke Shell Container
docker compose exec app sh

### Restart Container
docker compose restart app

### Refresh Cache Setelah Perubahan Code atau .env
docker compose exec app php artisan optimize:clear
docker compose exec app php artisan optimize

### Matikan Container
docker compose down

### Cek log
docker compose logs -f --tail=30 app
