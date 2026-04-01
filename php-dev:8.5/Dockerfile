FROM debian:bookworm-slim

ARG PHP_VERSION=8.5

ENV DEBIAN_FRONTEND=noninteractive

# ── Base packages + Sury PHP repository ───────────────────────────────────────
RUN set -eux; \
    apt-get update; \
    apt-get upgrade -y; \
    apt-get install -y --no-install-recommends \
        apt-transport-https ca-certificates wget gnupg lsb-release locales openssl; \
    localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8; \
    DEBIAN_CODE_NAME=$(lsb_release -c -s); \
    wget -O /usr/share/keyrings/sury-archive-keyring.gpg https://packages.sury.org/php/apt.gpg; \
    echo "deb [signed-by=/usr/share/keyrings/sury-archive-keyring.gpg] https://packages.sury.org/php/ ${DEBIAN_CODE_NAME} main" \
        | tee /etc/apt/sources.list.d/php.list; \
    apt-get update

# ── Apache + PHP + extensions ─────────────────────────────────────────────────
# php-common bundles: bcmath, calendar, exif, ftp, gettext, pcntl, shmop,
#   sockets, sysvmsg, sysvsem, sysvshm, tokenizer
# php-xml bundles:    dom, simplexml, xml, xmlreader, xmlwriter, xsl
# opcache is built-in as of PHP 8.5 (no separate package)
RUN set -eux; \
    mkdir -p /usr/share/man/man1; \
    apt-get install -y --no-install-recommends \
        procps unzip bzip2 patch git vim iputils-ping \
        apache2 libapache2-mod-php${PHP_VERSION} \
        php${PHP_VERSION}-apcu \
        php${PHP_VERSION}-bcmath \
        php${PHP_VERSION}-bz2 \
        php${PHP_VERSION}-cli \
        php${PHP_VERSION}-common \
        php${PHP_VERSION}-curl \
        php${PHP_VERSION}-gd \
        php${PHP_VERSION}-gmp \
        php${PHP_VERSION}-imagick \
        php${PHP_VERSION}-intl \
        php${PHP_VERSION}-ldap \
        php${PHP_VERSION}-mbstring \
        php${PHP_VERSION}-memcached \
        php${PHP_VERSION}-mongodb \
        php${PHP_VERSION}-mysql \
        php${PHP_VERSION}-pgsql \
        php${PHP_VERSION}-readline \
        php${PHP_VERSION}-redis \
        php${PHP_VERSION}-soap \
        php${PHP_VERSION}-tidy \
        php${PHP_VERSION}-xdebug \
        php${PHP_VERSION}-xml \
        php${PHP_VERSION}-zip \
        php-pear; \
    update-alternatives --set php /usr/bin/php${PHP_VERSION}; \
    apt-get autoremove --purge -y; \
    apt-get clean; \
    rm -rf /var/lib/apt/lists/*

# ── Node.js LTS ────────────────────────────────────────────────────────────────
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && curl -fsSL https://deb.nodesource.com/setup_lts.x | bash - \
    && apt-get install -y nodejs \
    && rm -rf /var/lib/apt/lists/* \
    && npm install -g npm@latest

# ── Composer ──────────────────────────────────────────────────────────────────
COPY --from=composer:latest /usr/bin/composer /usr/local/bin/composer

# ── Apache ────────────────────────────────────────────────────────────────────
RUN a2enmod rewrite headers ssl php${PHP_VERSION}

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
RUN cat > /etc/php/${PHP_VERSION}/apache2/conf.d/99-custom.ini <<'EOF'
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
opcache.enable                  = 1
opcache.enable_cli              = 1
opcache.memory_consumption      = 128
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files   = 20000
opcache.validate_timestamps     = 1
opcache.revalidate_freq         = 0
opcache.fast_shutdown           = 1

; ── APCu ──────────────────────────────────────────────────────────────────────
apc.enable_cli = 1
EOF

# ── Xdebug configuration ──────────────────────────────────────────────────────
RUN cat > /etc/php/${PHP_VERSION}/apache2/conf.d/99-xdebug.ini <<'EOF'
[xdebug]
xdebug.mode               = debug,develop
xdebug.start_with_request = yes
xdebug.client_host        = host.docker.internal
xdebug.client_port        = 9003
xdebug.idekey             = PHPSTORM
xdebug.log_level          = 0
xdebug.max_nesting_level  = 512
EOF

# ── Apache entrypoint ─────────────────────────────────────────────────────────
ENV APACHE_RUN_USER=www-data
ENV APACHE_RUN_GROUP=www-data

WORKDIR /var/www/html

EXPOSE 80 443

CMD ["apache2ctl", "-D", "FOREGROUND"]
