# Local dev stack — Caddy + PHP-FPM + MySQL

A lightweight local development stack built with Docker Compose:

- **`webserver`** (Caddy) — terminates HTTPS, serves static files, proxies PHP via FastCGI
- **`php-fpm`** (PHP-FPM) — runs all your PHP apps (version selectable, default 8.4)
- **MySQL 8**
- **phpMyAdmin**
- **Redis**

Sites are served over trusted HTTPS on local domains (`*.local`).

## Architecture

```
browser ──https──▶ webserver ──fastcgi:9000──▶ php-fpm
                     │                            │
                     └── static files             └── /var/www  (= ./projects)
                                               database (mysql:8), redis, phpmyadmin
```

Both `webserver` and `php-fpm` mount your project directory (`DOCUMENT_ROOT`,
default `./projects`) at `/var/www`, so Caddy and PHP-FPM agree on every script
path. One `php-fpm` image serves every site (`Dockerfile-php-fpm`).

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) with Compose v2
- [mkcert](https://github.com/FiloSottile/mkcert) — for trusted local HTTPS certificates

## First-time setup

1. **Create your env file:**

   ```shell
   cp sample.env .env
   ```

   Edit `.env` and change the MySQL credentials and any host ports that
   conflict with services already running on your machine.

2. **Generate a local certificate** and trust the CA (one time, asks for your password):

   ```shell
   mkcert -install
   mkcert -cert-file config/caddy/certs/local.pem -key-file config/caddy/certs/local-key.pem \
     app.local localhost 127.0.0.1 ::1
   ```

   Include every domain you reference in the Caddyfile. Re-run the second
   command whenever you add a new domain.

3. **Map your domains to localhost** in `/etc/hosts`:

   ```
   127.0.0.1 app.local
   ```

4. **Create your Caddyfile** from the sample and add your sites (see below):

   ```shell
   cp config/caddy/Caddyfile.sample config/caddy/Caddyfile
   ```

5. **Start the stack:**

   ```shell
   docker compose up -d
   ```

## Configure sites

Put each project in a subdirectory of `DOCUMENT_ROOT` (default `./projects`),
then add a site block to `config/caddy/Caddyfile`. A reusable `(local)` snippet
provides compression and the mkcert TLS certificate.

```caddy
# Plain PHP site, docroot = projects/mysite
app.local {
	import local
	root * /var/www/mysite
	php_fastcgi php-fpm:9000
	file_server
	log {
		output file /var/log/caddy/app.log
	}
}

# Laravel (or any framework with a public/ docroot)
laravel.local {
	import local
	root * /var/www/myapp/public
	php_fastcgi php-fpm:9000
	file_server
}
```

Reload Caddy after editing: `docker compose restart webserver`.

## Common commands

```shell
docker compose up -d              # start
docker compose down               # stop
docker compose build php-fpm      # rebuild the PHP image after editing the Dockerfile
docker compose restart webserver  # reload after editing the Caddyfile
docker compose exec php-fpm bash  # shell into the PHP container (composer, artisan, etc.)
docker compose logs -f webserver  # tail web server logs
```

Run Composer / Laravel Artisan inside the `php-fpm` container, e.g.:

```shell
docker compose exec php-fpm bash -c "cd /var/www/myapp && composer install"
docker compose exec php-fpm bash -c "cd /var/www/myapp && php artisan migrate"
```

## Configuration

All settings live in `.env` (copy from `sample.env`). Key values:

- `DOCUMENT_ROOT` — host path mounted into the containers (default `./projects`)
- `PHP_INI` — custom `php.ini` (default `./config/php/php.ini`)
- `HOST_MACHINE_*_PORT` — host port mappings (web 80/443, MySQL 3306, PMA 8080, Redis 6379)
- `MYSQL_ROOT_PASSWORD` — MySQL root password

### Database

This stack runs MySQL with a single **root** account (it's local dev only). Set
`MYSQL_ROOT_PASSWORD` in `.env` and connect as `root` — from an app use host
`database` (the service name), not `localhost`; from the host machine connect to
`127.0.0.1:3306`.

There is no auto-created database or app user. Create databases yourself however
you like:

- in **phpMyAdmin** (http://localhost:8080), or any MySQL client, or
- by dropping one or more `*.sql` files into `./config/initdb/` — they run
  automatically the **first time** the container initializes (i.e. when
  `./data/mysql` is empty).

MySQL data persists in `./data/mysql`. `config/mysql/my.cnf` keeps the legacy
`mysql_native_password` plugin enabled for compatibility with older connectors.

### Switching PHP version

The `php-fpm` image is built from a selectable base. Set `PHPVERSION` in `.env`
(supported: `8.1`, `8.2`, `8.3`, `8.4`) then rebuild:

```shell
# .env  ->  PHPVERSION=8.2
docker compose build php-fpm && docker compose up -d
```

The built image is tagged with the PHP version
(`<project>-php-fpm:<version>`), so each version you build is kept as its own
image. Switching back and forth between versions reuses the already-built image
instead of rebuilding. List them with `docker images | grep php-fpm`.

### PHP extensions & tools

The image (`Dockerfile-php-fpm`) bundles these PHP extensions: `pdo_mysql`,
`mysqli`, `pdo_sqlite`, `bcmath`, `zip`, `intl`, `mbstring`, `gettext`,
`calendar`, `exif`, `gd`, `redis`, `imagick`, and `xdebug`.

It also includes **`wkhtmltopdf`** (official build with patched Qt, for HTML→PDF
rendering) and **Composer**. Add more there and rebuild with
`docker compose build php-fpm`.

### phpMyAdmin

http://localhost:8080 — user `root`, password from `MYSQL_ROOT_PASSWORD` in `.env`.

### Xdebug

Xdebug 3 is installed. Enable it by uncommenting the settings in
`config/php/php.ini` and rebuilding/restarting the `php-fpm` container. The
container reaches your IDE at `host.docker.internal:9003`.

## Production note

This stack is for **local development only**. It uses an internal CA and exposes
admin tooling — do not deploy it as-is, and always change the default credentials
in `.env`.
```