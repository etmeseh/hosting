# Running hosting in Docker

This is an optional, unofficial way to try out or run this project in a
container instead of directly on a server. It does not modify `install.sh`
in any way — it just runs the unmodified script inside a systemd-enabled
Ubuntu 24.04 container, because `install.sh` manages every service
(nginx, mariadb, tor, postfix, dovecot, php-fpm, ...) through `systemctl`.

## Requirements

- Docker Engine or Docker Desktop with Compose v2 (`docker compose`)
- The container needs `--privileged` and access to a cgroup v2 hierarchy
  for systemd to run, which `docker/docker-compose.yml` already sets up
- Same host sizing as a normal install: at least 4 GB RAM / 1 CPU for a
  personal/test instance (see the main README for production sizing)
- The first boot compiles ImageMagick and four PHP versions from source,
  so expect the initial `docker compose up` to take a while (well over
  30 minutes depending on the host)

## Usage

From the repository root:

```
docker compose -f docker/docker-compose.yml up -d --build
```

Watch the first-boot install:

```
docker compose -f docker/docker-compose.yml exec hosting tail -f /var/log/hosting-firstboot.log
```

Installation only runs once, guarded by `docker/hosting-firstboot.service`
(a systemd oneshot unit) and a marker file at `/var/www/.hosting-firstboot.done`.
Restarting or recreating the container will not re-run it as long as the
`www-data` volume still has that marker.

Once it's done, get your credentials and onion address:

```
docker compose -f docker/docker-compose.yml exec hosting cat /root/hosting-credentials.txt
```

## Vanity onion address / custom passwords

By default the container generates a random onion address and random
passwords, same as `install.sh --non-interactive` on its own. To set a
vanity prefix or your own passwords, export the matching `HOSTING_*`
variable before starting the container (these are the same variables
`install.sh` itself reads):

```
HOSTING_VANITY_PREFIX=myprefix HOSTING_ADMIN_PASS=mypassword \
  docker compose -f docker/docker-compose.yml up -d --build
```

Available overrides: `HOSTING_VANITY_PREFIX`, `HOSTING_VANITY_THREADS`,
`HOSTING_DB_PASS`, `HOSTING_PMA_PASS`, `HOSTING_ADMIN_PASS`,
`HOSTING_SKIP_BINARIES`. These only take effect on first boot, before
`/var/www/.hosting-firstboot.done` exists.

## Persistence

`docker/docker-compose.yml` defines named volumes for the state that
matters across container recreation:

- `mysql-data` -> `/var/lib/mysql`
- `www-data` -> `/var/www` (site files, mail, and the first-boot marker)
- `tor-data` -> `/var/lib/tor` (your onion address's private key)
- `home-data` -> `/home` (hosting account home directories)

## Updating

To pull in upstream changes and rebuild:

```
docker compose -f docker/docker-compose.yml exec hosting bash -c 'cd /root/hosting && ./update.sh'
```
