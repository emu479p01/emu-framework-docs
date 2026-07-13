# Install with Docker

## Purpose

Run immutable EmuFramework images with persistent database storage.

## Audience

Docker operators and framework administrators.

## Prerequisites

Docker Engine with Compose and access to `ghcr.io`.

## Procedure

1. Obtain `docker-compose.yml` from the [framework repository](https://github.com/emu479p01/emu-framework/blob/master/docker-compose.yml). Cloning the repository is not required.
2. Create `.env` beside `docker-compose.yml`:

   ```env
   EMU_VERSION=0.0.1.1
   EMU_UPDATER_TOKEN=replace-with-a-random-secret-at-least-24-characters
   PORT=3399
   EMU_SECURE_COOKIES=true
   ```

   The token is created by the operator; it is not downloaded from GitHub. Use the same token for the app and updater, and never commit `.env`.
3. Run:

   ```sh
   docker compose pull
   docker compose up -d
   docker compose ps
   ```

4. Open `http://localhost:3399`.

The Compose file supplies the app with `EMU_DEPLOYMENT_MODE=docker`, connects it to the internal updater at `http://updater:3400`, and mounts the named `emu-data` volume at `/data`.

## Image source

Both images are published to the GitHub Container Registry (`ghcr.io`) under the `emu479p01` org, listed at [Your Packages](https://github.com/emu479p01?tab=packages&repo_name=emu-framework):

```text
ghcr.io/emu479p01/emu-framework:<version>          # app
ghcr.io/emu479p01/emu-framework-updater:<version>  # updater
```

`docker-compose.yml` resolves `<version>` from `EMU_VERSION` in `.env`, so `docker compose pull` fetches both images at that tag. No account or authentication is required to pull, since the packages are public. Pulling manually without Compose looks like:

```sh
docker pull ghcr.io/emu479p01/emu-framework:<version>
docker pull ghcr.io/emu479p01/emu-framework-updater:<version>
```

To use a specific version instead of the latest, set `EMU_VERSION` accordingly before running `docker compose pull`.

## Docker Desktop without cloning

Pulling an image alone is not a complete installation. `docker pull` downloads an image but does not create the app, updater, network, environment variables, or persistent volume.

Create the shared network and volume first:

```powershell
docker network create emu-network
docker volume create emu-data
docker network inspect emu-network
docker volume inspect emu-data
```

If Docker reports that the network or volume already exists, it can be reused, but verify it belongs to this EmuFramework installation before continuing.

The app container must have:

```text
Image: ghcr.io/emu479p01/emu-framework:<version>
Container name: emu-framework
Restart policy: Unless stopped
Port: 3399 -> 3399
Network: emu-network
Volume: emu-data -> /data (Read/Write)
NODE_ENV=production
EMU_APP_TITLE=EmuFramework
EMU_DEPLOYMENT_MODE=docker
EMU_UPDATER_URL=http://updater:3400
EMU_UPDATER_TOKEN=<same token as updater>
EMU_DB_PATH=/data/data.db
EMU_DESIGNER_DB_PATH=/data/designer.db
```

The updater container must use the matching updater image, the same token, the same `emu-network` network, the `emu-data` volume, and a Docker socket mount:

```text
Image: ghcr.io/emu479p01/emu-framework-updater:<version>
Container name: emu-framework-updater
Restart policy: Unless stopped
Network: emu-network
Network alias: updater
Volume: emu-data -> /data (Read/Write)
Docker socket: /var/run/docker.sock -> /var/run/docker.sock (Read/Write)
EMU_UPDATER_TOKEN=<same token as app>
EMU_APP_CONTAINER=emu-framework
EMU_IMAGE_REPOSITORY=ghcr.io/emu479p01/emu-framework
EMU_UPDATE_STATE_PATH=/data/update-status.json
```

Do not publish updater port `3400` to the host; it is an internal service. The updater's container name can be anything, but it must carry the network alias `updater`, because the app reaches it at `http://updater:3400`. If the containers were created without a network, connect them afterward:

```powershell
docker network connect emu-network emu-framework
docker network connect --alias updater emu-network emu-framework-updater
```

After changing environment variables, recreate the app container. Restarting an existing container does not add new environment variables.

### Verify app-to-updater connectivity

```powershell
docker exec emu-framework node -e "fetch('http://updater:3400/').then(r=>console.log(r.status)).catch(e=>{console.error(e);process.exit(1)})"
```

An HTTP `404` response confirms the network and DNS alias work; the updater has no `GET /` route, but the app reached the service successfully. Check logs for either side if this fails:

```powershell
docker logs --tail 100 emu-framework
docker logs --tail 100 emu-framework-updater
```

## Existing installation and data migration

Before removing an existing container, inspect its mounts:

```sh
docker inspect <app-container> --format '{{json .Mounts}}'
```

The database files should be mounted at `/data`. A named volume such as `emu-data` can be reused safely when recreating the container. Do not run `docker compose down -v` unless deleting all application data is intentional. Compose names its containers `emuframework-app` and `emuframework-updater`; manually created containers are commonly named `emu-framework` and `emu-framework-updater`. List actual names with:

```sh
docker ps -a --format "table {{.Names}}\t{{.Image}}"
```

Find the volume mounted at `/data` for the app container:

```sh
docker inspect emu-framework --format '{{range .Mounts}}{{if eq .Destination "/data"}}{{.Name}}{{end}}{{end}}'
```

A long hexadecimal result is normally an anonymous volume. Verify it before continuing:

```sh
docker volume inspect <SOURCE_VOLUME>
```

Stop both the app and the updater before copying, so the SQLite database and its WAL files are in a consistent state:

```sh
docker stop emu-framework emu-framework-updater
docker volume create emu-data
```

Copy the data. This command refuses to run if `emu-data` already contains files, preventing an accidental merge of two installations:

```sh
docker run --rm --mount type=volume,src=<SOURCE_VOLUME>,dst=/source,readonly --mount type=volume,src=emu-data,dst=/target alpine:3.20 sh -c 'if [ -n "$(find /target -mindepth 1 -maxdepth 1 -print -quit)" ]; then echo "ERROR: emu-data is not empty"; exit 1; fi; cp -a /source/. /target/'
```

Inspect the copied files before changing or removing anything:

```sh
docker run --rm --mount type=volume,src=emu-data,dst=/data,readonly alpine:3.20 sh -c "find /data -maxdepth 2 -type f -exec ls -ln {} ';'"
```

Expect to find `data.db`, `designer.db`, `backups/`, and `update-status.json`; the exact list depends on features previously used. Recreate both containers with `emu-data` mounted at `/data` — a running container's volume mount cannot be changed in place:

```sh
docker compose up -d --force-recreate
```

For manually managed containers, remove only the stopped containers (never the source volume yet) and create them again with `emu-data -> /data` for both app and updater. Confirm the mount on each:

```sh
docker inspect emu-framework --format '{{range .Mounts}}{{println .Name "->" .Destination}}{{end}}'
docker inspect emu-framework-updater --format '{{range .Mounts}}{{println .Name "->" .Destination}}{{end}}'
```

Before removing the old volume, verify: the app opens at `http://localhost:3399`; existing tables and transactions are intact; reports and installed fonts still work; the backup list still appears; the app can reach the updater; and **Check for updates** succeeds without `fetch failed`. Keep the old volume until all of this is confirmed and a separate backup exists:

```sh
docker volume rm <SOURCE_VOLUME>
```

## Separate database volumes

By default `data.db` and `designer.db` share the single `emu-data` volume mounted at `/data`. To isolate them on separate volumes:

1. Add a second named volume, for example `emu-designer-data`, to `docker-compose.yml` and mount it on the app container alongside the existing one:

   ```text
   emu-data:/data
   emu-designer-data:/designer-data
   ```

2. Set:

   ```env
   EMU_DB_PATH=/data/data.db
   EMU_DESIGNER_DB_PATH=/designer-data/designer.db
   ```

3. Before recreating the container, create a verified backup and inspect the existing mount:

   ```sh
   docker inspect <app-container> --format '{{json .Mounts}}'
   ```

4. Create the new volume and copy `designer.db` (and its `-wal`/`-shm` files if present) into it:

   ```sh
   docker volume create emu-designer-data
   docker run --rm -v emu-data:/from -v emu-designer-data:/to alpine sh -c 'cp -a /from/designer.db* /to/'
   ```

5. Recreate the app container so it picks up the new environment variables:

   ```sh
   docker compose up -d --force-recreate app
   ```

6. Verify `data.db` remains in `emu-data` and `designer.db` now lives in `emu-designer-data`, then confirm the app starts and reads both correctly before removing `designer.db` from the original volume.

Do not run `docker compose down -v` during this process; it deletes named volumes.

## Expected result

The app uses the `emu-data` volume. The updater has no public port.

## Common errors

- Do not commit `.env`.
- The updater mounts `/var/run/docker.sock`, which grants host-level Docker control. Disable the updater and use the manual procedure if that risk is unacceptable.
- `EMU_UPDATER_URL` is normally `http://updater:3400`; it is a Docker service name, not a public URL and not `localhost`.
- `Deployment: unsupported` means the **app** container did not receive `EMU_DEPLOYMENT_MODE=docker`; setting the variable only on the updater is insufficient.
- If the app cannot resolve `updater`, the two containers are not on the same user-defined network.
- If port `3399` is already allocated, stop the old app container or choose another host port, for example `3401:3399`.

## Related topics

[Docker operations](docker-operations.md) · [Framework update](framework-update.md)
