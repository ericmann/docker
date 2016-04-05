# Dockerfile written by Eric Mann <eric@eamann.com>
#
# Work derived from official PHP Docker Library:
# Copyright (c) 2014-2016 Docker, Inc.

FROM alpine:3.3

# phpize deps
RUN apk add --no-cache --virtual .phpize-deps \
		autoconf \
		file \
		g++ \
		gcc \
		libc-dev \
		make \
		pkgconf \
		re2c

# persistent / runtime deps
RUN apk add --no-cache --virtual .persistent-deps \
		ca-certificates \
		curl \
		libssl1.0 \
		libxml2-dev \
		libxml2 \
		libpng-dev \
		libmcrypt-dev \
		zlib

# ensure www-data user exists
RUN set -x \
	&& addgroup -g 82 -S www-data \
	&& adduser -u 82 -D -S -G www-data www-data
# 82 is the standard uid/gid for "www-data" in Alpine

ENV PHP_INI_DIR /usr/local/etc/php
RUN mkdir -p $PHP_INI_DIR/conf.d

ENV PHP_EXTRA_CONFIGURE_ARGS --enable-fpm --with-fpm-user=www-data --with-fpm-group=www-data --enable-maintainer-zts --enable-pthreads

ENV GPG_KEYS 1A4E8B7277C42E53DBA9C7B9BCAA30EA9C0D5763

ENV PHP_VERSION 7.0.5
ENV PHP_FILENAME php-7.0.5.tar.xz

RUN set -xe \
	&& apk add --no-cache --virtual .build-deps \
		curl-dev \
		gnupg \
		libedit-dev \
		openssl-dev \
		sqlite-dev \
	&& curl -fSL "http://php.net/get/$PHP_FILENAME/from/this/mirror" -o "$PHP_FILENAME" \
	&& curl -fSL "http://php.net/get/$PHP_FILENAME.asc/from/this/mirror" -o "$PHP_FILENAME.asc" \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& for key in $GPG_KEYS; do \
		gpg --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
	done \
	&& gpg --batch --verify "$PHP_FILENAME.asc" "$PHP_FILENAME" \
	&& rm -r "$GNUPGHOME" "$PHP_FILENAME.asc" \
	&& mkdir -p /usr/src \
	&& tar -Jxf "$PHP_FILENAME" -C /usr/src \
	&& mv "/usr/src/php-$PHP_VERSION" /usr/src/php \
	&& rm "$PHP_FILENAME" \
	&& cd /usr/src/php \
	&& ./configure \
		--with-config-file-path="$PHP_INI_DIR" \
		--with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
		$PHP_EXTRA_CONFIGURE_ARGS \
		--disable-cgi \
		--enable-mysqlnd \
		--enable-mbstring \
		--with-curl \
		--with-libedit \
		--with-openssl \
		--with-zlib \
	&& make -j"$(getconf _NPROCESSORS_ONLN)" \
	&& make install \
	&& { find /usr/local/bin /usr/local/sbin -type f -perm +0111 -exec strip --strip-all '{}' + || true; } \
	&& make clean \
	&& runDeps="$( \
		scanelf --needed --nobanner --recursive /usr/local \
			| awk '{ gsub(/,/, "\nso:", $2); print "so:" $2 }' \
			| sort -u \
			| xargs -r apk info --installed \
			| sort -u \
	)" \
	&& apk add --virtual .php-rundeps $runDeps \
	&& apk del .build-deps

COPY docker-php-ext-* /usr/local/bin/

RUN set -ex \
	&& cd /usr/local/etc \
	&& if [ -d php-fpm.d ]; then \
		# for some reason, upstream's php-fpm.conf.default has "include=NONE/etc/php-fpm.d/*.conf"
		sed 's!=NONE/!=!g' php-fpm.conf.default | tee php-fpm.conf > /dev/null; \
		cp php-fpm.d/www.conf.default php-fpm.d/www.conf; \
	else \
		# PHP 5.x don't use "include=" by default, so we'll create our own simple config that mimics PHP 7+ for consistency
		mkdir php-fpm.d; \
		cp php-fpm.conf.default php-fpm.d/www.conf; \
		{ \
			echo '[global]'; \
			echo 'include=etc/php-fpm.d/*.conf'; \
		} | tee php-fpm.conf; \
	fi \
	&& { \
		echo '[global]'; \
		echo 'error_log = /proc/self/fd/2'; \
		echo; \
		echo '[www]'; \
		echo '; if we send this to /proc/self/fd/1, it never appears'; \
		echo 'access.log = /proc/self/fd/2'; \
		echo; \
		echo 'clear_env = no'; \
		echo; \
		echo '; Ensure worker stdout and stderr are sent to the main error log.'; \
		echo 'catch_workers_output = yes'; \
	} | tee php-fpm.d/docker.conf \
	&& { \
		echo '[global]'; \
		echo 'daemonize = no'; \
		echo; \
		echo '[www]'; \
		echo 'listen = [::]:9000'; \
	} | tee php-fpm.d/zz-docker.conf

RUN extensions=" \
        gd \
        mysqli \
        soap \
        mcrypt \
        mbstring \
    "; \
    apk update && apk add --no-cache --virtual .extension-deps \
		rsync \
    && docker-php-ext-install $extensions \
    && apk del .extension-deps \

RUN peclDeps=" \
		libmemcached-dev \
	"; \
	apk update && apk add --no-cache --virtual .pecl-deps \
	libmemcached-dev \
#	&& pecl install redis && echo extension=redis.so > $PHP_INI_DIR/conf.d/ext-redis.ini \
#	&& pecl install memcached && echo extension=memcached.so > $PHP_INI_DIR/conf.d/ext-memcached.ini \
#	&& pecl install memcache && echo extension=memcache.so > $PHP_INI_DIR/conf.d/ext-memcache.ini \
	&& pecl install apcu && echo extension=apcu.so > $PHP_INI_DIR/conf.d/ext-apcu.ini \
	&& apk del .pecl-deps

RUN apk update && apk add --no-cache --virtual .memcached-deps \
		git \
		libmemcached-dev \
	&& git clone https://github.com/php-memcached-dev/php-memcached.git && cd php-memcached \
	&& git checkout php7 \
	&& phpize \
	&& ./configure --disable-memcached-sasl \
	&& make && make install \
	&& echo extension=memcached.so > $PHP_INI_DIR/conf.d/ext-memcached.ini \
	&& curl -fSL "https://github.com/websupport-sk/pecl-memcache/archive/fdbd46bbc6f53ed6e024521895e142cbfc9b3340/memcache-3.0.9-fdbd46b.tar.gz" -o "memcache.tar.gz" \
	&& mkdir -p /usr/src/php-memcache \
    && tar -xf "memcache.tar.gz" -C /usr/src/php-memcache --strip-components=1 \
    && cd /usr/src/php-memcache && phpize \
    && ./configure \
    && make && make install \
    && echo extension=memcache.so > $PHP_INI_DIR/conf.d/ext-memcache.ini \
	&& apk del .memcached-deps

MAINTAINER Eric Mann

WORKDIR /var/www/html

EXPOSE 9070
CMD ["php-fpm"]