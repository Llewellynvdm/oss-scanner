# syntax=docker/dockerfile:1
# Use the PHP/Apache base selected for Joomla's official 6.2 image. The CMS
# itself comes from the checked-out branch so PHP, assets and tests agree.
FROM node:24-trixie-slim AS node
FROM composer:2 AS composer
FROM php:8.4-apache-trixie

COPY --from=node /usr/local/bin/node /usr/local/bin/node
COPY --from=node /usr/local/lib/node_modules/npm /usr/local/lib/node_modules/npm
COPY --from=composer /usr/bin/composer /usr/local/bin/composer

# Install the extensions needed by the CMS and its locked development packages.
# The local database lets the installer run without a second container.
RUN set -eux; \
    apt-get update; \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        ca-certificates curl git python3 rsync unzip \
        mariadb-client mariadb-server \
        libfreetype6-dev libgmp-dev libicu-dev libjpeg-dev libldap2-dev \
        libpng-dev libwebp-dev libzip-dev; \
    rm -rf /var/lib/apt/lists/*; \
    docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp; \
    docker-php-ext-configure ldap --with-libdir="lib/$(dpkg-architecture -qDEB_BUILD_MULTIARCH)"; \
    docker-php-ext-install -j2 bcmath exif gd gmp intl ldap mysqli pdo_mysql zip; \
    ln -s ../lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm; \
    ln -s ../lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx

ENV JOOMLA_WEBROOT=/src \
    JOOMLA_DB_SOCKET=/run/mysqld/mysqld.sock

# These paths can be overridden together for a disposable installation.
RUN set -eux; \
    sed -i 's@/var/www/html@${JOOMLA_WEBROOT}@' /etc/apache2/sites-available/000-default.conf; \
    sed -i 's/^Listen 80$/Listen 127.0.0.1:80/' /etc/apache2/ports.conf; \
    printf '%s\n' '<Directory "${JOOMLA_WEBROOT}">' \
        '    Options FollowSymLinks' '    AllowOverride All' '    Require all granted' '</Directory>' \
        '<DirectoryMatch "/\.git(/|$)">' '    Require all denied' '</DirectoryMatch>' \
        > /etc/apache2/conf-available/joomla.conf; \
    a2enmod rewrite; \
    a2enconf joomla; \
    printf '%s\n' 'mysqli.default_socket = ${JOOMLA_DB_SOCKET}' > "$PHP_INI_DIR/conf.d/joomla-db.ini"

WORKDIR /src
COPY . /src

# The npm install hook builds the media assets. Cypress is only used by the browser suite.
RUN COMPOSER_ALLOW_SUPERUSER=1 composer install --prefer-dist --no-interaction --no-progress \
    && composer check-platform-reqs \
    && CYPRESS_INSTALL_BINARY=0 npm ci --no-audit --no-fund \
    && rm -rf /root/.cache/composer /root/.npm

# The installer can remove installation/ on release revisions. Exercise it on a disposable
# copy of the built checkout to preserve the source and let the check run more than once.
COPY --chmod=0755 <<'BASH' /usr/local/bin/joomla-smoke
#!/usr/bin/env bash
set -euo pipefail

work=$(mktemp -d)
db_pid=
http_pid=
cleanup() {
    for pid in "$http_pid" "$db_pid"; do
        if [[ -n $pid ]]; then
            kill "$pid" 2>/dev/null || true
            wait "$pid" || true
        fi
    done
    rm -rf -- "$work"
}
trap cleanup EXIT

chmod 0711 "$work"
install -d -o mysql -g mysql "$work/db" "$work/run"
export JOOMLA_DB_SOCKET="$work/run/mysql.sock"
export JOOMLA_WEBROOT="$work/site"
rsync -a --chown=www-data:www-data --exclude=/.git --exclude=/node_modules /src/ "$JOOMLA_WEBROOT/"
cd "$JOOMLA_WEBROOT"

# Keep the database and its credentials within this installation.
mariadb-install-db --no-defaults --user=mysql --datadir="$work/db" --skip-test-db >/dev/null
mariadbd --no-defaults --user=mysql --datadir="$work/db" --socket="$JOOMLA_DB_SOCKET" \
    --pid-file="$work/run/mysql.pid" --skip-networking &
db_pid=$!
mariadb-admin --no-defaults --user=root --socket="$JOOMLA_DB_SOCKET" --wait=15 ping
db_password=$(php -r 'echo bin2hex(random_bytes(24));')
mariadb --no-defaults --user=root --socket="$JOOMLA_DB_SOCKET" <<SQL
CREATE DATABASE joomla CHARACTER SET utf8mb4;
CREATE USER 'joomla'@'localhost' IDENTIFIED BY '$db_password';
GRANT ALL ON joomla.* TO 'joomla'@'localhost';
SQL

runuser -u www-data -- php installation/joomla.php install --no-interaction \
    --site-name='Joomla smoke test' --admin-user='Local Administrator' --admin-username=localadmin \
    --admin-password="$(php -r 'echo "Aa1!", bin2hex(random_bytes(20));')" --admin-email=admin@example.org \
    --db-type=mysqli --db-host=localhost --db-user=joomla --db-pass="$db_password" --db-name=joomla
runuser -u www-data -- php cli/joomla.php cache:clean --help > /dev/null

# The site name distinguishes an installed frontend from an installer or error page.
apache2-foreground &
http_pid=$!
curl --fail --silent --show-error --retry 10 --retry-connrefused --retry-delay 1 --max-time 15 \
    http://127.0.0.1/ -o "$work/home.html"
grep -Fq 'Joomla smoke test' "$work/home.html"
BASH

RUN --network=none ./libraries/vendor/bin/phpunit --testsuite Unit && joomla-smoke

CMD ["/bin/bash"]
