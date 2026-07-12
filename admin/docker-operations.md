# Operate Docker

## Common commands

```sh
docker compose up -d
docker compose stop
docker compose start
docker compose ps
docker compose logs --tail=200 app
docker compose logs --tail=200 updater
docker compose pull
```

Use `docker compose down` only when the containers and network should be removed; the named volume remains. Never add `-v` unless deleting all application data is intentional.

## Verify the updater

The updater is intentionally not exposed on a host port. It is reached only by the app over the Compose network:

```text
http://updater:3400
```

Check both services and their logs:

```sh
docker compose ps
docker compose logs --tail=100 app
docker compose logs --tail=100 updater
```

Confirm that the app, rather than only the updater, has the required settings:

```sh
docker compose exec app printenv EMU_DEPLOYMENT_MODE
docker compose exec app printenv EMU_UPDATER_URL
```

Expected values are `docker` and `http://updater:3400`. Recreate the app after changing `.env`:

```sh
docker compose up -d --force-recreate app
```

## Docker Desktop checks

When containers were created manually in Docker Desktop, verify that both containers share a user-defined network and that the app has a `/data` volume. The updater must have access to `/var/run/docker.sock`; this grants host-level Docker control and should be treated as a high-privilege component.

## Manual rollback

1. Keep the verified pre-update backup.
2. Set `EMU_VERSION` in `.env` to the previous known-good version.
3. Run `docker compose pull && docker compose up -d`.
4. Restore databases only when a documented migration requires it; container rollback does not alter database files.

## Related topics

[Docker installation](docker-install.md) · [Database storage](database-storage.md)
