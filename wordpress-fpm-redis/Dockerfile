FROM wordpress:php7.1-fpm

RUN apt-get update && apt-get install -y \
        libicu-dev \
        libmcrypt-dev \
        libmagickwand-dev \
        libsodium-dev \
        --no-install-recommends && rm -r /var/lib/apt/lists/* \

    && pecl install redis-3.1.1 imagick-3.4.3 libsodium-1.0.6 \
    && docker-php-ext-enable redis imagick libsodium \
    && docker-php-ext-install -j$(nproc) exif gettext intl mcrypt sockets zip
