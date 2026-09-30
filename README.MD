#!/usr/bin/env bash
set -e

# ==============================================================================
# CONFIGURATION - Adjust these values before running
# ==============================================================================
PROJECT_DIR="laravel-app"
REPO_URL="https://github.com/username/repository.git"
DOMAIN="your-domain.com"
APP_PORT="8000"

# ==============================================================================
# 1. CLONE & DEPENDENCIES
# ==============================================================================
cd /var/www

if [ ! -d "$PROJECT_DIR" ]; then
    git clone "$REPO_URL" "$PROJECT_DIR"
fi

cd "$PROJECT_DIR"

# Install production vendor dependencies
composer install --optimize-autoloader --no-dev

# ==============================================================================
# 2. ENVIRONMENT FILE SETUP (.env)
# ==============================================================================
if [ ! -f .env ]; then
    cp .env.example .env
    sed -i "s|APP_URL=.*|APP_URL=https://${DOMAIN}|g" .env
    sed -i "s|APP_ENV=.*|APP_ENV=production|g" .env
    sed -i "s|APP_DEBUG=.*|APP_DEBUG=false|g" .env
fi

# ==============================================================================
# 3. DOCKER BUILD & CONTAINER LAUNCH
# ==============================================================================
docker compose build --no-cache
docker compose up -d

# Wait briefly for containers to initialize
sleep 5

# ==============================================================================
# 4. LARAVEL CONTAINER INITIALIZATION
# ==============================================================================
docker compose exec -T app php artisan key:generate --force
docker compose exec -T app php artisan storage:link
docker compose exec -T app php artisan migrate --force
docker compose exec -T app php artisan optimize:clear
docker compose exec -T app php artisan optimize

# ==============================================================================
# 5. REVERSE PROXY SETUP (NGINX HOST)
# ==============================================================================
cat <<EOF > /etc/nginx/sites-available/${DOMAIN}.conf
server {
    listen 80;
    server_name ${DOMAIN};

    client_max_body_size 100M;

    location / {
        proxy_pass http://127.0.0.1:${APP_PORT};
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF

ln -sf /etc/nginx/sites-available/${DOMAIN}.conf /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx

# ==============================================================================
# 6. VERIFICATION
# ==============================================================================
curl -I http://127.0.0.1:${APP_PORT}
