FROM php:8.5-apache

ENV DEBIAN_FRONTEND=noninteractive

# ── System libraries ──────────────────────────────────────────────────────────
RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates curl git gnupg unzip wget \
        libbz2-dev \
        libcurl4-openssl-dev \
        libffi-dev \
        libfreetype6-dev \
        libgmp-dev \
        libicu-dev \
        libjpeg62-turbo-dev \
        libldap2-dev \
        libmagickwand-dev \
        libmemcached-dev \
        libpng-dev \
        libpq-dev \
        libreadline-dev \
        libsodium-dev \
        libtidy-dev \
        libwebp-dev \
        libxml2-dev \
        libxslt-dev \
        libzip-dev \
        zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*

# ── Configure extensions that need flags ─────────────────────────────────────
RUN docker-php-ext-configure gd \
        --with-freetype \
        --with-jpeg \
        --with-webp \
    && docker-php-ext-configure ldap \
        --with-libdir="lib/$(dpkg-architecture --query DEB_BUILD_MULTIARCH)"

# ── Built-in extensions ───────────────────────────────────────────────────────
RUN docker-php-ext-install -j"$(nproc)" \
    bcmath \
    bz2 \
    calendar \
    curl \
    dom \
    exif \
    ffi \
    ftp \
    gd \
    gettext \
    gmp \
    intl \
    ldap \
    mbstring \
    mysqli \
    opcache \
    pcntl \
    pdo \
    pdo_mysql \
    pdo_pgsql \
    pgsql \
    readline \
    shmop \
    simplexml \
    soap \
    sockets \
    sodium \
    sysvmsg \
    sysvsem \
    sysvshm \
    tidy \
    xml \
    xmlreader \
    xmlwriter \
    xsl \
    zip

# ── PECL extensions ───────────────────────────────────────────────────────────
# igbinary + msgpack must be installed first — redis/memcached depend on them
RUN pecl install igbinary \
    && docker-php-ext-enable igbinary

RUN pecl install msgpack \
    && docker-php-ext-enable msgpack

RUN pecl install apcu \
    && docker-php-ext-enable apcu

# Redis with igbinary + msgpack serialiser support
RUN pecl install --configureoptions \
        'enable-redis-igbinary="yes" enable-redis-msgpack="yes" enable-redis-lzf="no" enable-redis-zstd="no"' \
        redis \
    && docker-php-ext-enable redis

# Memcached with igbinary + msgpack support
RUN pecl install --configureoptions \
        'enable-memcached-igbinary="yes" enable-memcached-msgpack="yes" enable-memcached-json="yes"' \
        memcached \
    && docker-php-ext-enable memcached

RUN pecl install mongodb \
    && docker-php-ext-enable mongodb

RUN pecl install imagick \
    && docker-php-ext-enable imagick

RUN pecl install xdebug \
    && docker-php-ext-enable xdebug

# ── Node.js LTS ───────────────────────────────────────────────────────────────
RUN curl -fsSL https://deb.nodesource.com/setup_lts.x | bash - \
    && apt-get install -y nodejs \
    && rm -rf /var/lib/apt/lists/* \
    && npm install -g npm@latest

# ── Composer ──────────────────────────────────────────────────────────────────
COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer

# ── Apache ────────────────────────────────────────────────────────────────────
RUN a2enmod rewrite headers ssl

RUN cat > /etc/apache2/sites-available/000-default.conf <<'EOF'
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/public

    <Directory /var/www/html/public>
        AllowOverride All
        Require all granted
        Options -Indexes +FollowSymLinks
    </Directory>

    ErrorLog  ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
EOF

# ── PHP configuration ─────────────────────────────────────────────────────────
RUN cat > /usr/local/etc/php/conf.d/99-custom.ini <<'EOF'
; ── Error handling ────────────────────────────────────────────────────────────
display_errors         = On
display_startup_errors = On
error_reporting        = E_ALL
log_errors             = On

; ── Limits ────────────────────────────────────────────────────────────────────
memory_limit          = 512M
max_execution_time    = 120
max_input_time        = 120
post_max_size         = 100M
upload_max_filesize   = 100M
max_file_uploads      = 50

; ── Misc ──────────────────────────────────────────────────────────────────────
date.timezone         = UTC
default_charset       = UTF-8

; ── OPcache ───────────────────────────────────────────────────────────────────
; Enabled in dev for realistic performance; timestamps checked on every request
opcache.enable                 = 1
opcache.enable_cli             = 1
opcache.memory_consumption     = 128
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files  = 20000
opcache.validate_timestamps    = 1
opcache.revalidate_freq        = 0
opcache.fast_shutdown          = 1

; ── APCu ──────────────────────────────────────────────────────────────────────
apc.enable_cli = 1
EOF

RUN cat > /usr/local/etc/php/conf.d/99-xdebug.ini <<'EOF'
[xdebug]
xdebug.mode              = debug,develop
xdebug.start_with_request = yes
xdebug.client_host       = host.docker.internal
xdebug.client_port       = 9003
xdebug.idekey            = PHPSTORM
xdebug.log_level         = 0
; Raise if you hit the limit on deeply nested calls (e.g. large Laravel apps)
xdebug.max_nesting_level = 512
EOF

WORKDIR /var/www/html

EXPOSE 80 443
