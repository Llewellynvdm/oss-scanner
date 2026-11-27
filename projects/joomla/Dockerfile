# syntax=docker/dockerfile:1
# Joomla! CMS.
#
# /src    the joomla-cms git checkout (the repository named in project.yaml), with its locked Composer and npm
#         dependencies installed, its media assets built from media_source/ into media/, and a working site
#         installed into it by the CMS's own console installer. Apache serves /src through mod_php with AllowOverride
#         All, so a renamed htaccess.txt applies as it does in production. The database is MariaDB on a unix socket.
#
# Start the services with `joomla-start`. The site is then at http://127.0.0.1/, the administrator at
# http://127.0.0.1/administrator/ and the web services API at http://127.0.0.1/api/index.php/v1/. `joomla-stop` stops
# them. threat_model.md lists the accounts, the API tokens and the other helpers.
#
# The PHP and Apache base is the one Joomla's official 6.2 image is built on. Node and Composer come from their
# official images, at the versions the CMS's lock files expect.
FROM node:24-trixie-slim AS node
FROM composer:2 AS composer
FROM php:8.4-apache-trixie

COPY --from=node /usr/local/bin/node /usr/local/bin/node
COPY --from=node /usr/local/lib/node_modules/npm /usr/local/lib/node_modules/npm
COPY --from=composer /usr/bin/composer /usr/local/bin/composer

# MariaDB supplies the server and the client. Keep the libraries used by the PHP extensions and discard their build
# headers, as the official image does.
RUN set -eux; \
    apt-get update; \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        git mariadb-server unzip; \
    savedAptMark="$(apt-mark showmanual)"; \
    apt-get install -y --no-install-recommends \
        libfreetype-dev libicu-dev libjpeg-dev libldap-dev \
        libpng-dev libwebp-dev libzip-dev; \
    docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp; \
    docker-php-ext-configure ldap --with-libdir="lib/$(dpkg-architecture -qDEB_BUILD_MULTIARCH)"; \
    docker-php-ext-install -j"$(nproc)" exif gd intl ldap mysqli pdo_mysql zip; \
    apt-mark auto '.*' > /dev/null; \
    apt-mark manual $savedAptMark > /dev/null; \
    ldd "$(php-config --extension-dir)"/*.so \
        | awk '/=> \/(usr\/)?lib\// { so = $(NF-1); sub("^/(usr/)?", "", so); print "*" so }' \
        | sort -u | xargs -r dpkg-query --search \
        | awk 'sub(":$", "", $1) { print $1 }' | sort -u | xargs -r apt-mark manual; \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
    apt-get dist-clean; \
    ln -s ../lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm; \
    ln -s ../lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx

# Production-like PHP: errors are logged to Apache's error log, not shown in responses; OPcache picks up an edited
# file at once, so a patch to /src takes effect on the next request; the upload limits allow media manager tests.
# MySQL clients reach MariaDB over its unix socket when the host is `localhost`, as the installed site does.
RUN { echo 'display_errors=Off'; \
      echo 'log_errors=On'; \
      echo 'error_reporting=E_ALL'; \
      echo 'memory_limit=256M'; \
      echo 'upload_max_filesize=32M'; \
      echo 'post_max_size=32M'; \
      echo 'opcache.validate_timestamps=1'; \
      echo 'opcache.revalidate_freq=0'; \
      echo 'sendmail_path=/usr/sbin/sendmail -t -i'; \
      echo 'mysqli.default_socket=/run/mysqld/mysqld.sock'; \
      echo 'pdo_mysql.default_socket=/run/mysqld/mysqld.sock'; \
    } > "$PHP_INI_DIR/conf.d/oss-scanner.ini"

# Apache serves /src on the loopback interface only, with .htaccess enabled. The checkout's .git and the directories
# a Joomla release does not ship (the scanner's Dockerfile, build tooling, asset sources, tests, node_modules) stay
# private. The base image links Apache's logs to the console, which a daemonised Apache does not have; make them real
# files, so PHP and Apache errors end up in /var/log/apache2/error.log.
RUN set -eux; \
    a2enmod rewrite headers expires; \
    sed -i 's@/var/www/html@/src@' /etc/apache2/sites-available/000-default.conf; \
    sed -i 's/^Listen 80$/Listen 127.0.0.1:80/' /etc/apache2/ports.conf; \
    printf '%s\n' 'ServerName 127.0.0.1' \
        '<Directory /src>' '    Options -Indexes +FollowSymLinks' '    AllowOverride All' '    Require all granted' '</Directory>' \
        '<DirectoryMatch "^/src/(\.git|\.oss-scanner|build|media_source|node_modules|tests)(/|$)">' \
        '    Require all denied' '</DirectoryMatch>' \
        > /etc/apache2/conf-available/joomla.conf; \
    a2enconf joomla; \
    rm -f /var/log/apache2/*.log

# There is no mail server. This sendmail writes each email Joomla sends (password resets, registration, multi-factor
# codes, notifications) to /var/mail/joomla/, so reproductions can read the links and codes in them.
RUN mkdir -p /var/mail/joomla && chown www-data:www-data /var/mail/joomla \
    && printf '%s\n' '#!/bin/sh' 'cat > "/var/mail/joomla/$(date +%Y%m%d-%H%M%S)-$$.eml"' > /usr/sbin/sendmail \
    && chmod 0755 /usr/sbin/sendmail

# Helpers (there is no init system in the container):
#   joomla-start      starts MariaDB and Apache; safe to run again
#   joomla-stop       stops both
#   as-www-data       runs a command as www-data, the user Apache runs as and the owner of /src
#   joomla-console    cli/joomla.php as www-data
#   joomla-phpunit    PHPUnit as www-data, from /src, with the checkout's phpunit.xml
#   joomla-api-token  prints a user's web services API token, giving them one if they have none
COPY --chmod=0755 <<'SH' /usr/local/bin/joomla-start
#!/bin/sh
set -e
# Files root creates in these directories (for example by running cli/joomla.php as root) would stop the site, which
# runs as www-data, from writing there. Hand them back first.
chown -R www-data:www-data /src/administrator/cache /src/administrator/logs /src/cache /src/tmp /src/images /var/mail/joomla
mkdir -p /run/mysqld && chown mysql:mysql /run/mysqld
if ! mariadb-admin ping --silent 2>/dev/null; then
    "$(command -v mariadbd-safe || command -v mysqld_safe)" >/dev/null 2>&1 &
    until mariadb-admin ping --silent 2>/dev/null; do sleep 0.5; done
fi
if ! { [ -s /var/run/apache2/apache2.pid ] && kill -0 "$(cat /var/run/apache2/apache2.pid)" 2>/dev/null; }; then
    apachectl start
fi
SH
COPY --chmod=0755 <<'SH' /usr/local/bin/joomla-stop
#!/bin/sh
apachectl stop 2>/dev/null
pid=$(cat /run/mysqld/mysqld.pid 2>/dev/null || true)
mariadb-admin shutdown 2>/dev/null
while [ -n "$pid" ] && kill -0 "$pid" 2>/dev/null; do sleep 0.2; done
true
SH
COPY --chmod=0755 <<'SH' /usr/local/bin/as-www-data
#!/bin/sh
exec runuser -u www-data -- env HOME=/tmp "$@"
SH
COPY --chmod=0755 <<'SH' /usr/local/bin/joomla-console
#!/bin/sh
cd /src && exec as-www-data php /src/cli/joomla.php "$@"
SH
COPY --chmod=0755 <<'SH' /usr/local/bin/joomla-phpunit
#!/bin/sh
cd /src && exec as-www-data /src/libraries/vendor/bin/phpunit "$@"
SH
COPY --chmod=0755 <<'PHP' /usr/local/bin/joomla-api-token
#!/usr/bin/env php
<?php
// Print a user's web services API token, as the "Joomla API Token" tab of their profile shows it. A user who has
// none gets one, as saving their profile would give them. For example:
//   curl -H "Authorization: Bearer $(joomla-api-token superuser)" http://127.0.0.1/api/index.php/v1/users
\define('_JEXEC', 1);
require '/src/configuration.php';
$config   = new JConfig();
$username = $argv[1] ?? '';
if ($username === '') {
    fwrite(STDERR, "usage: joomla-api-token <username>\n");
    exit(2);
}
// With the host `localhost`, mysqli uses the unix socket from php.ini, as the site does.
$db = new mysqli($config->host, $config->user, $config->password, $config->db);
$db->set_charset('utf8mb4');
$users    = $config->dbprefix . 'users';
$profiles = $config->dbprefix . 'user_profiles';
$query = $db->prepare("SELECT id FROM $users WHERE username = ?");
$query->bind_param('s', $username);
$query->execute();
$id = (int) $query->get_result()->fetch_column();
if (!$id) {
    fwrite(STDERR, "no user named $username\n");
    exit(1);
}
$query = $db->prepare("SELECT profile_key, profile_value FROM $profiles WHERE user_id = ? AND profile_key LIKE 'joomlatoken.%'");
$query->bind_param('i', $id);
$query->execute();
$profile = array_column($query->get_result()->fetch_all(MYSQLI_ASSOC), 'profile_value', 'profile_key');
if (empty($profile['joomlatoken.token'])) {
    $seed  = base64_encode(random_bytes(32));
    $query = $db->prepare("DELETE FROM $profiles WHERE user_id = ? AND profile_key LIKE 'joomlatoken.%'");
    $query->bind_param('i', $id);
    $query->execute();
    $query = $db->prepare("INSERT INTO $profiles (user_id, profile_key, profile_value, ordering) VALUES (?, 'joomlatoken.token', ?, 1), (?, 'joomlatoken.enabled', '1', 2)");
    $query->bind_param('isi', $id, $seed, $id);
    $query->execute();
    $profile = ['joomlatoken.token' => $seed, 'joomlatoken.enabled' => '1'];
}
if ((int) ($profile['joomlatoken.enabled'] ?? 0) !== 1) {
    fwrite(STDERR, "the API token of $username is disabled in their profile\n");
    exit(1);
}
echo base64_encode('sha256:' . $id . ':' . hash_hmac('sha256', base64_decode($profile['joomlatoken.token']), $config->secret)), "\n";
PHP

# scanner contract: the checkout lives inside the image, at /src. As on a production site, the web server owns the
# Joomla directory, so the site can write its caches, logs and uploads and the files an extension installation
# creates. The scanner works as root, so git is told to trust the directory.
COPY --chown=www-data:www-data . /src
WORKDIR /src
RUN git config --system --add safe.directory /src

# The locked PHP dependencies, including the dev ones (PHPUnit, PHP CS Fixer, PHPCS, PHPStan), then the locked npm
# dependencies: Joomla's `install` script builds the media assets from media_source/ into media/. Cypress's
# browser is not downloaded: the System (browser) test suite needs a browser and a site elsewhere, and does not run here.
RUN as-www-data composer install --prefer-dist --no-interaction --no-progress \
    && as-www-data composer check-platform-reqs --no-interaction \
    && as-www-data env CYPRESS_INSTALL_BINARY=0 npm ci --no-audit --no-fund \
    && rm -rf /tmp/.cache /tmp/.composer /tmp/.config /tmp/.local /tmp/.npm

# Install a site into /src with the CMS's own console installer, on MariaDB over the unix socket, then add one user
# per core group, so access-control reproductions can act as each. Every password is "<login>-password".
#   superuser      Super Users (the installer's administrator account)
#   administrator  Administrator
#   manager        Manager
#   publisher      Publisher
#   editor         Editor
#   author         Author
#   registered     Registered
#   (anonymous)    Public: the Guest group, no login
# Databases: `joomla` for the site (user joomla / joomla, table prefix jos_) and `joomla_test` for the Integration test
# suite, whose phpunit.xml points at it. Two settings differ from a stock site so that every account can use the API
# with the permissions of its group: Registered and Manager (and the groups under them) may log in to the API, which a
# stock site allows Super Users alone, and every group may hold an API token (see joomla-api-token).
RUN --network=none <<'EOF'
#!/bin/bash
set -euo pipefail
joomla-start
mariadb <<'SQL'
CREATE DATABASE joomla CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE joomla_test CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'joomla'@'localhost' IDENTIFIED BY 'joomla';
GRANT ALL ON joomla.* TO 'joomla'@'localhost';
GRANT ALL ON joomla_test.* TO 'joomla'@'localhost';
SQL
as-www-data php installation/joomla.php install --no-interaction \
    --site-name='Joomla OSS Scanner' --admin-user='Super User' --admin-username=superuser \
    --admin-password=superuser-password --admin-email=superuser@example.org \
    --db-type=mysqli --db-host=localhost --db-user=joomla --db-pass=joomla --db-name=joomla --db-prefix=jos_
mariadb joomla <<'SQL'
UPDATE jos_assets SET rules = REPLACE(REPLACE(rules,
    '"core.login.api":{"8":1}', '"core.login.api":{"8":1,"2":1,"6":1}'),
    '"core.api":{"8":1}', '"core.api":{"8":1,"2":1,"6":1}') WHERE id = 1;
UPDATE jos_extensions SET params = '{"allowedUserGroups":["2","3","4","5","6","7","8"]}'
    WHERE type = 'plugin' AND folder = 'user' AND element = 'token';
SQL
mariadb -N joomla -e "SELECT rules FROM jos_assets WHERE id = 1" | grep '"core.login.api":{"8":1,"2":1,"6":1}' > /dev/null
for spec in administrator:Administrator manager:Manager publisher:Publisher editor:Editor author:Author registered:Registered; do
    IFS=: read -r login group <<< "$spec"
    joomla-console user:add --username="$login" --name="$group" --email="$login@example.org" \
        --password="$login-password" --usergroup="$group"
done
joomla-console user:list
# Point the Integration suite at this MariaDB and skip its LDAP tests, as tests/Integration/README.md describes.
sed -e 's@JTEST_DB_HOST" value="mysql"@JTEST_DB_HOST" value="localhost"@' \
    -e 's@JTEST_DB_NAME" value="test_joomla"@JTEST_DB_NAME" value="joomla_test"@' \
    -e 's@JTEST_DB_USER" value="joomla_ut"@JTEST_DB_USER" value="joomla"@' \
    -e 's@JTEST_DB_PASSWORD" value="joomla_ut"@JTEST_DB_PASSWORD" value="joomla"@' \
    -e 's@JTEST_LDAP_HOST" value="openldap"@JTEST_LDAP_HOST" value=""@' phpunit.xml.dist > phpunit.xml
chown www-data:www-data phpunit.xml
joomla-stop
EOF

# Check, offline, that the site, the administrator, the API and the console work, and that Apache keeps the
# checkout's .git private and runs configuration.php instead of serving it. A failing check fails the build: the
# scanner would get an image it cannot use. A failing test does not: the branch is upmerged daily, and one red test
# must not stop the scan.
RUN --network=none <<'EOF'
#!/bin/bash
set -euo pipefail
joomla-start
status() { curl -s -o /dev/null -w '%{http_code}' "$@"; }
curl -fsSL http://127.0.0.1/ | grep 'Joomla OSS Scanner' > /dev/null                      # the installed site, not the installer
curl -fsSL http://127.0.0.1/administrator/index.php | grep 'name="username"' > /dev/null   # the administrator login form
[[ "$(status http://127.0.0.1/api/index.php/v1/content/articles)" =~ ^40[13]$ ]]          # the API refuses anonymous requests
curl -fsS -H "Authorization: Bearer $(joomla-api-token superuser)" http://127.0.0.1/api/index.php/v1/users | grep '"superuser"' > /dev/null
curl -fsS -H "Authorization: Bearer $(joomla-api-token editor)" http://127.0.0.1/api/index.php/v1/content/articles | grep '"data"' > /dev/null
test "$(status http://127.0.0.1/.git/HEAD)" = 403
test "$(status http://127.0.0.1/configuration.php)" = 200 && test -z "$(curl -s http://127.0.0.1/configuration.php)"
curl -fsS http://127.0.0.1/LICENSE.txt | cmp - /src/LICENSE.txt
joomla-console cache:clean
joomla-phpunit --testsuite Unit || echo "WARNING: the Unit suite reported failures"
joomla-phpunit --testsuite Integration || echo "WARNING: the Integration suite reported failures"
joomla-stop
# Start the scan with an empty error log and mailbox: what is in them now came from the build.
: > /var/log/apache2/error.log
rm -f /var/mail/joomla/*.eml
EOF

# Keep build products out of the checkout's `git status`.
RUN printf '.oss-scanner/\n' >> /src/.git/info/exclude && git status --short

CMD ["bash"]
