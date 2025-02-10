FROM php:8.1-apache AS builder

LABEL vendor="Mautic"
LABEL maintainer="Mautic core team <>"

# Lower optimization level for compilation
ENV CFLAGS="-O0"

# Install the dependency required for compiling the mcrypt extension,
# install mcrypt via PECL, enable the extension, and clean up.
RUN apt-get update && \
    apt-get install -y libmcrypt-dev && \
    pecl install mcrypt-1.0.5 && \
    docker-php-ext-enable mcrypt && \
    rm -rf /var/lib/apt/lists/*


RUN apt-get update && apt-get install -y \
    libc-client-dev \
    libkrb5-dev 

RUN docker-php-ext-configure imap --with-kerberos --with-imap-ssl
RUN docker-php-ext-install imap


# Install PHP extensions
RUN apt-get update && apt-get install --no-install-recommends -y \
    ca-certificates \
    build-essential  \
    git \
    curl \
    libcurl4-gnutls-dev \
    libc-client-dev \
    libkrb5-dev \
    libmcrypt-dev \
    libssl-dev \
    libxml2-dev \
    libzip-dev \
    libjpeg-dev \
    libmagickwand-dev \
    libpng-dev \
    libgif-dev \
    libtiff-dev \
    libz-dev \
    libpq-dev \
    imagemagick \
    graphicsmagick \
    libwebp-dev \
    libjpeg62-turbo-dev \
    libxpm-dev \
    libaprutil1-dev \
    libicu-dev \
    libfreetype6-dev \
    libonig-dev \
    librabbitmq-dev \
    unzip \
    nodejs \
    npm \
    libc-client-dev

RUN apt-get update && apt-get install -y \
    libkrb5-dev

# Configure and install the PHP intl extension
RUN docker-php-ext-configure intl

RUN curl -L -o /tmp/amqp.tar.gz "https://github.com/php-amqp/php-amqp/archive/refs/tags/v2.1.1.tar.gz" \
    && mkdir -p /usr/src/php/ext/amqp \
    && tar -C /usr/src/php/ext/amqp -zxvf /tmp/amqp.tar.gz --strip 1 \
    && rm /tmp/amqp.tar.gz

# Configure the gd extension.
RUN docker-php-ext-configure gd --with-freetype --with-jpeg


# Configure the opcache extension.
RUN docker-php-ext-configure opcache --enable-opcache



# Install the PHP extensions.
RUN docker-php-ext-install intl 
RUN docker-php-ext-install mbstring 
RUN docker-php-ext-install mysqli 
RUN docker-php-ext-install curl 
RUN docker-php-ext-install pdo_mysql 
RUN docker-php-ext-install zip 
RUN docker-php-ext-install bcmath 
# Installed by default
# RUN docker-php-ext-install sockets 
RUN docker-php-ext-install exif 
RUN docker-php-ext-install gd 
RUN docker-php-ext-install opcache

# Install the amqp extension via PECL.
RUN pecl install amqp && docker-php-ext-enable amqp

# Enable the PHP extensions.
RUN docker-php-ext-enable intl 
RUN docker-php-ext-enable mbstring
RUN docker-php-ext-enable mysqli 
RUN docker-php-ext-enable curl 
RUN docker-php-ext-enable pdo_mysql 
RUN docker-php-ext-enable zip 
RUN docker-php-ext-enable bcmath
RUN docker-php-ext-enable exif
RUN docker-php-ext-enable amqp
RUN docker-php-ext-enable gd
RUN docker-php-ext-enable imap
RUN docker-php-ext-enable opcache



# Install composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/bin --filename=composer

RUN echo "memory_limit = -1" > /usr/local/etc/php/php.ini

# Define Mautic version by package tag
ARG MAUTIC_VERSION=5.x-dev

RUN cd /opt && \
    COMPOSER_ALLOW_SUPERUSER=1 COMPOSER_PROCESS_TIMEOUT=10000 composer create-project mautic/recommended-project:${MAUTIC_VERSION} mautic --no-interaction && \
    rm -rf /opt/mautic/var/cache/js && \
    find /opt/mautic/node_modules -mindepth 1 -maxdepth 1 -not \( -name 'jquery' -or -name 'vimeo-froogaloop2' \) | xargs rm -rf

FROM php:8.1-apache

COPY --from=builder /usr/local/lib/php/extensions /usr/local/lib/php/extensions
COPY --from=builder /usr/local/etc/php/conf.d/ /usr/local/etc/php/conf.d/

COPY --from=builder --chown=www-data:www-data /opt/mautic /var/www/html

# Install PHP extensions requirements and other dependencies
RUN apt-get update && apt-get install --no-install-recommends -y \
    unzip libwebp-dev libzip-dev libfreetype6-dev libjpeg62-turbo-dev libpng-dev libc-client-dev librabbitmq4 \
    mariadb-client supervisor cron \
    && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
    && rm -rf /var/lib/apt/lists/* \
    && rm /etc/cron.daily/*

# Setting PHP properties
ENV PHP_INI_VALUE_DATE_TIMEZONE='UTC' \
    PHP_INI_VALUE_MEMORY_LIMIT=512M \
    PHP_INI_VALUE_UPLOAD_MAX_FILESIZE=512M \
    PHP_INI_VALUE_POST_MAX_FILESIZE=512M \
    PHP_INI_VALUE_MAX_EXECUTION_TIME=300

COPY ./common/php.ini /usr/local/etc/php/php.ini

ENV APACHE_DOCUMENT_ROOT=/var/www/html/docroot

RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf
RUN sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf

# Enable Apache Rewrite Module
RUN a2enmod rewrite

COPY ./common/docker-entrypoint.sh /entrypoint.sh
COPY ./common/entrypoint_mautic_web.sh /entrypoint_mautic_web.sh
COPY ./common/entrypoint_mautic_cron.sh /entrypoint_mautic_cron.sh
COPY ./common/entrypoint_mautic_worker.sh /entrypoint_mautic_worker.sh

# Apply necessary permissions
RUN ["chmod", "+x", "/entrypoint.sh", "/entrypoint_mautic_web.sh", "/entrypoint_mautic_cron.sh", "/entrypoint_mautic_worker.sh"]

# Setting worker env vars
ENV DOCKER_MAUTIC_WORKERS_CONSUME_EMAIL=2 \
    DOCKER_MAUTIC_WORKERS_CONSUME_HIT=2 \
    DOCKER_MAUTIC_WORKERS_CONSUME_FAILED=2

COPY ./common/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# Define Mautic volumes to persist data
VOLUME /var/www/html/config
VOLUME /var/www/html/var/logs
VOLUME /var/www/html/docroot/media

WORKDIR /var/www/html/docroot

ENV DOCKER_MAUTIC_ROLE=mautic_web \
    DOCKER_MAUTIC_RUN_MIGRATIONS=false \
    DOCKER_MAUTIC_LOAD_TEST_DATA=false

ENTRYPOINT ["/entrypoint.sh"]

CMD ["apache2-foreground"]
