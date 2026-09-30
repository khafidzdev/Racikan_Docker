# base jika menggunakan FrankenPHP dengan PHP 8.4 
FROM dunglas/frankenphp:php8.4-alpine

# install ekstensi php yang diperlukan
RUN install-php-extensions \
    pdo_mysql \
    mbstring \
    exif \
    pcntl \
    bcmath \
    gd \
    zip \
    intl \
    opcache \
    redis

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /app

COPY . .

ENV COMPOSER_ALLOW_SUPERUSER=1

RUN composer install --optimize-autoloader --no-dev --no-interaction

RUN mkdir -p storage/logs \
    storage/framework/cache \
    storage/framework/sessions \
    storage/framework/views \
    bootstrap/cache \
    && chown -R www-data:www-data /app \
    && chmod -R 775 storage bootstrap/cache

COPY docker/php/Caddyfile /etc/caddy/Caddyfile
COPY docker/php/php.ini /usr/local/etc/php/conf.d/99-custom.ini

EXPOSE 80 443 443/udp

CMD ["frankenphp", "run", "--config", "/etc/caddy/Caddyfile"]