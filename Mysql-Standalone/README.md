# MySQL 8.4 LTS: Standalone

A single MySQL server in one container, using the official `mysql` image.

> **Standalone means one instance.** There is no replication and no high availability.
> If the host or container is down, the database is down.

> **Status:** these files were written following the official image documentation.
> They have **not** been tested yet. Run through "Verify" below before relying on them.

## Files

| File | Purpose |
|------|---------|
| `compose.yaml` | The service definition |
| `.env.example` | Settings template. Copy to `.env` |

## Configure and start

```bash
cp .env.example .env
# Edit .env: set MYSQL_ROOT_PASSWORD (and optionally MYSQL_DATABASE / MYSQL_USER / MYSQL_PASSWORD)
chmod 600 .env
docker compose up -d
```

Important: the root password and the optional database/user are applied **only on the first start**,
when the data volume is empty. Changing them in `.env` later does not change an existing database.
To change a password later, use SQL:

```bash
docker compose exec mysql sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD"'
```
```sql
ALTER USER 'root'@'%' IDENTIFIED BY 'new-password';
ALTER USER 'root'@'localhost' IDENTIFIED BY 'new-password';
```
Then update `.env` so the healthcheck and backup commands keep working.

## Verify

```bash
docker compose ps                  # STATUS should become "healthy" (can take up to a minute on first start)
docker compose logs mysql | tail   # look for "ready for connections"
docker compose exec mysql sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT VERSION();"'
```

## Access

By default the port is published on `127.0.0.1:3306` only.

- From inside the container (admin shell):
  `docker compose exec mysql sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD"'`
- From the host: `mysql -h 127.0.0.1 -P 3306 -u <user> -p`
- From other containers: put them on the same Compose network (or an external shared network)
  and connect to host `mysql` (the service name), port `3306`. No published port is needed.
- From other servers: set `MYSQL_BIND_ADDRESS=0.0.0.0` in `.env`, then restrict access with a firewall.
  Note: Docker-published ports can bypass `ufw`/`firewalld` rules, so check how your firewall is set up.

Use the root account for administration only. Applications should use their own user
(`MYSQL_USER`) with only the privileges they need.

## Optional features

### Extra server settings (`conf.d`)

1. Create `conf.d/custom.cnf`:
   ```ini
   [mysqld]
   max_connections = 300
   innodb_buffer_pool_size = 1G
   slow_query_log = ON
   long_query_time = 2
   ```
2. Uncomment `- ./conf.d:/etc/mysql/conf.d:ro` in `compose.yaml`.
3. `docker compose up -d` (recreates the container) and confirm in SQL: `SHOW VARIABLES LIKE 'max_connections';`

`innodb_buffer_pool_size` defaults to 128 MB, which is small for most real workloads.
Size it for your server (commonly 50-70% of RAM available to MySQL). If you set `mem_limit`
in `compose.yaml`, base the value on that limit.

### TLS

MySQL 8.4 already enables TLS with an auto-generated self-signed certificate.
Check from SQL: `SHOW SESSION STATUS LIKE 'Ssl_cipher';` (non-empty = encrypted).

To use your own certificate:

1. Place `ca.pem`, `server-cert.pem`, `server-key.pem` in `./certs/`.
2. The key must be readable by the MySQL user inside the image (UID 999):
   `sudo chown 999:999 certs/server-key.pem && sudo chmod 600 certs/server-key.pem`
3. Uncomment the `./certs` volume and the three `--ssl-*` lines in `compose.yaml`.
4. `docker compose up -d`, then verify with `SHOW SESSION STATUS LIKE 'Ssl_cipher';`.
5. Only after TLS works for all clients, optionally uncomment `--require-secure-transport=ON`.

### Init scripts

Files in `./initdb/` (`*.sql`, `*.sql.gz`, `*.sh`) run in alphabetical order, **once**, on first start with an empty
data volume. Uncomment the `./initdb` volume to use it.

### Legacy clients

MySQL 8.4 disables `mysql_native_password` by default. If an old client cannot use `caching_sha2_password`,
uncomment `--mysql-native-password=ON`. Prefer upgrading the client when possible.

## Backup and restore

This is a **logical backup** (a SQL dump). It restores to the moment the dump was taken, not to an arbitrary point in time.

Backup (all databases, consistent for InnoDB):

```bash
docker compose exec -T mysql sh -c 'exec mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" \
  --all-databases --single-transaction --routines --triggers --events' \
  | gzip > mysql-backup-$(date +%F).sql.gz
```

Restore (into a running, healthy instance):

```bash
gunzip -c mysql-backup-YYYY-MM-DD.sql.gz \
  | docker compose exec -T mysql sh -c 'exec mysql -uroot -p"$MYSQL_ROOT_PASSWORD"'
```

Notes:
- A dump of `--all-databases` also contains user accounts, so restoring it overwrites accounts,
  including the root password, with the ones from the dump.
- Store backups **off the host**, and test a restore regularly. An untested backup is not a backup.
- You will see the warning "Using a password on the command line interface can be insecure". It is expected here.

## Common commands

```bash
docker compose up -d                 # start
docker compose ps                    # status and health
docker compose logs -f mysql         # follow logs
docker compose restart mysql         # restart
docker compose stop                  # stop (data kept)
docker compose pull && docker compose up -d   # after changing MYSQL_VERSION in .env
docker compose down                  # remove container, data volume is kept
docker compose down -v               # DANGER: also deletes the data volume (all data)
```

## Production considerations

- **No HA.** One host, one instance. Use Primary/Replica (next in this toolkit) for read scaling and a warm copy,
  but note that replication alone is not automatic failover.
- **Backups are your responsibility.** Schedule the dump above (cron or systemd timer) and copy it off the host.
  A named volume on the same disk is not a backup.
- **Upgrades:** change `MYSQL_VERSION` within the same series (8.4.x) after reading the release notes, and take a backup first.
  Do not jump between major/LTS series without following the official upgrade guide.
- **Network exposure:** keep `127.0.0.1` unless remote access is required. If exposed, use a firewall and TLS.
- **Root account:** the image creates `root` with host `%`. This is fine while the port is bound to localhost,
  but do not expose the port widely while using root.
- **Passwords:** keep `.env` out of Git, with permissions `600`. Do not reuse the example values.
- **Resources:** set `innodb_buffer_pool_size` (and optionally `mem_limit`). Defaults are conservative.
- **Disk:** monitor free space on the Docker data directory. A full disk can corrupt writes and stop MySQL.
