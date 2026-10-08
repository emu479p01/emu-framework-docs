# Install with Docker

## Purpose

Run immutable EmuFramework images with persistent database, attachment, and archive storage.

## Audience

Docker operators and framework administrators.

## Prerequisites

Docker Engine with Compose and access to `ghcr.io`.

## Procedure

1. Obtain `docker-compose.yml` from the [framework repository](https://github.com/emu479p01/emu-framework/blob/master/docker-compose.yml). Cloning the repository is not required.
2. Create `.env` beside `docker-compose.yml`:

   ```env
   EMU_VERSION=1.4.0
   EMU_UPDATER_TOKEN=replace-with-a-random-secret-at-least-24-characters
   PORT=3399
   EMU_SECURE_COOKIES=true
   EMU_APP_TITLE=EmuFramework
   ```

   The token is created by the operator; it is not downloaded from GitHub. Use the same token for the app and updater, and never commit `.env`. `EMU_VERSION` is optional because the Compose file defaults to the release it ships with (`1.4.0`), but pinning it makes the installed version explicit. Use a stable `X.Y.Z` tag, never `latest`. `EMU_APP_TITLE` sets the product name shown in the browser title and on the login and setup pages.
3. Run:

   ```sh
   docker compose pull
   docker compose up -d
   docker compose ps
   ```

4. Read the one-time administrator setup code:

   ```sh
   docker compose logs app
   ```

5. Open `http://localhost:3399`, enter the code, choose the administrator username, and set a password of at least 12 characters.

The code expires after 15 minutes or ten failed attempts. Restart the app container to generate a new code:

```sh
docker compose restart app
docker compose logs --tail=100 app
```

The Compose file supplies the app with `EMU_DEPLOYMENT_MODE=docker`, connects it to the internal updater at `http://updater:3400`, and mounts three named volumes into both the app and the updater:

| Volume | Mount | Contents |
|---|---|---|
| `emu-data` | `/data` | `data.db`, `designer.db`, the secret key, backups, fonts, and update and restore status files |
| `emu-files` | `/files` | Live attachment files, selected with `EMU_FILE_STORAGE_PATH=/files` |
| `emu-archive` | `/archive` | The archive catalog and archived payloads, selected with `EMU_ARCHIVE_STORAGE_PATH=/archive` |

If `EMU_FILE_STORAGE_PATH` or `EMU_ARCHIVE_STORAGE_PATH` is not set, the app falls back to `/data/files` and `/data/archive` and the Storage Overview shows a warning that shared fallback storage is in use. Separate volumes keep attachment and archive growth away from the databases and let you place them on larger disks. See [Manage storage and archiving](storage-and-archive.md).

The default integration secret key is `/data/.emu-secret.key`, so it persists in the same named volume. It is not included in `.emubackup` exports; keep a separate secure copy after configuring SMTP. Set `EMU_SECRET_KEY_PATH` only when the replacement path is also mounted persistently.

### Overriding the volume and network names

Compose already gives the volumes and network explicit names (`emuframework-data`, `emuframework-files`, `emuframework-archive`, and `emuframework-network`), so backup, monitoring, migration, and external service connections can reference them reliably. To use different names — for example to avoid a collision with another deployment on the same host — set these before the first `docker compose up`:

```dotenv
EMU_VOLUME_NAME=mycompany-emu-data
EMU_FILES_VOLUME_NAME=mycompany-emu-files
EMU_ARCHIVE_VOLUME_NAME=mycompany-emu-archive
EMU_NETWORK_NAME=mycompany-emu-network
```

Changing these after the volume and network already exist does not rename them; set them before first creating the stack, or migrate data to the new volume as described below.

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

Pin the version with `EMU_VERSION` before running `docker compose pull`. Run the app and updater images at the same version. The web update replaces only the app container, so after it completes, set `EMU_VERSION` to the new version and run `docker compose pull` and `docker compose up -d` to bring the updater image to the same version.

## Docker Desktop without cloning

Pulling an image alone is not a complete installation. `docker pull` downloads an image but does not create the app, updater, network, environment variables, or persistent volume.

Create the shared network and volumes first:

```powershell
docker network create emu-network
docker volume create emu-data
docker volume create emu-files
docker volume create emu-archive
docker network inspect emu-network
docker volume inspect emu-data emu-files emu-archive
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
Volume: emu-files -> /files (Read/Write)
Volume: emu-archive -> /archive (Read/Write)
NODE_ENV=production
EMU_APP_TITLE=EmuFramework
EMU_DEPLOYMENT_MODE=docker
EMU_UPDATER_URL=http://updater:3400
EMU_UPDATER_TOKEN=<same token as updater>
EMU_DB_PATH=/data/data.db
EMU_DESIGNER_DB_PATH=/data/designer.db
EMU_SECRET_KEY_PATH=/data/.emu-secret.key
EMU_FILE_STORAGE_PATH=/files
EMU_ARCHIVE_STORAGE_PATH=/archive
```

After starting a manually created app container, run `docker logs --tail 100 emu-framework`, copy the one-time setup code, and complete **Administrator setup** at `http://localhost:3399`. Restart the container if the code expires.

The updater container must use the matching updater image, the same token, the same `emu-network` network, the `emu-data` volume, and a Docker socket mount:

```text
Image: ghcr.io/emu479p01/emu-framework-updater:<version>
Container name: emu-framework-updater
Restart policy: Unless stopped
Network: emu-network
Network alias: updater
Volume: emu-data -> /data (Read/Write)
Volume: emu-files -> /files (Read/Write)
Volume: emu-archive -> /archive (Read/Write)
Docker socket: /var/run/docker.sock -> /var/run/docker.sock (Read/Write)
EMU_UPDATER_TOKEN=<same token as app>
EMU_APP_CONTAINER=emu-framework
EMU_IMAGE_REPOSITORY=ghcr.io/emu479p01/emu-framework
EMU_UPDATE_STATE_PATH=/data/update-status.json
EMU_FILE_STORAGE_PATH=/files
EMU_ARCHIVE_STORAGE_PATH=/archive
```

The app and the updater must mount the same `/files` and `/archive` volumes. Before it stops the app, the updater compares the two containers' mounts and refuses to update if they differ (see [Update the framework](framework-update.md)).

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

An HTTP `404` response confirms the network and DNS alias work; the updater has no `GET /` route, but the app reached the service successfully. To check the updater's own health endpoint, request `/health` instead; it returns HTTP `200` with `{"ok":true,"busy":false}` when the updater is idle:

```powershell
docker exec emu-framework node -e "fetch('http://updater:3400/health').then(async r=>console.log(r.status, await r.text())).catch(e=>{console.error(e);process.exit(1)})"
```

Check logs for either side if this fails:

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

Expect to find `data.db`, `designer.db`, `backups/`, and `update-status.json`; the exact list depends on features previously used. An installation that already stored attachments before separate volumes were added may also contain `files/` and `archive/` directories. Recreate both containers with `emu-data` mounted at `/data` — a running container's volume mount cannot be changed in place:

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

## Add file and archive volumes to an existing installation

Installations created before v1.1.0 have only `emu-data`. They keep working after an update, with attachments and the archive stored in `/data/files` and `/data/archive`. To move to separate volumes:

1. Create a verified Full backup and a separate copy of the secret key.
2. Stop the stack with `docker compose stop`.
3. Replace your `docker-compose.yml` with the current one from the framework repository, or add the `emu-files:/files` and `emu-archive:/archive` volumes and the `EMU_FILE_STORAGE_PATH` and `EMU_ARCHIVE_STORAGE_PATH` variables to **both** services.
4. Create the volumes and copy the existing content, if any, before the stack starts. The example copies into the default Compose volume names:

   ```sh
   docker volume create emuframework-files
   docker volume create emuframework-archive
   docker run --rm -v emuframework-data:/from -v emuframework-files:/to alpine sh -c 'if [ -d /from/files ]; then cp -a /from/files/. /to/; fi'
   docker run --rm -v emuframework-data:/from -v emuframework-archive:/to alpine sh -c 'if [ -d /from/archive ]; then cp -a /from/archive/. /to/; fi'
   ```

5. Start the stack with `docker compose up -d --force-recreate`.
6. Open **Settings → System Maintenance → Storage Overview**, confirm the storage paths show `/files` and `/archive`, and confirm the fallback warning is gone. Open an existing attachment to confirm it still downloads.
7. After you have verified the result and kept a backup, remove the old `files` and `archive` directories from `/data` to reclaim the space.

## Database location

In production the app requires both `EMU_DB_PATH` and `EMU_DESIGNER_DB_PATH` to be under `/data/`, and it refuses to start otherwise. The updater snapshots `data.db` and `designer.db` from the `/data` volume before every update and restore, and backups read the same files. Keep both databases on the `emu-data` volume; placing `designer.db` on a separate volume is not supported. To give the databases more space, grow the disk that holds Docker volumes (see [Database storage](database-storage.md)). Attachments and the archive, which usually grow fastest, already use their own volumes.

Do not run `docker compose down -v`; it deletes named volumes.

## Expected result

The app uses the `emu-data`, `emu-files`, and `emu-archive` volumes, and the updater mounts the same three. The updater has no public port, and both containers report `healthy` in `docker compose ps`.

## Common errors

- Do not commit `.env`.
- The updater mounts `/var/run/docker.sock`, which grants host-level Docker control. Disable the updater and use the manual procedure if that risk is unacceptable.
- `EMU_UPDATER_URL` is normally `http://updater:3400`; it is a Docker service name, not a public URL and not `localhost`.
- `Deployment: unsupported` means the **app** container did not receive `EMU_DEPLOYMENT_MODE=docker`; setting the variable only on the updater is insufficient.
- If the app cannot resolve `updater`, the two containers are not on the same user-defined network.
- A warning in **Storage Overview** that shared fallback storage is in use means `EMU_FILE_STORAGE_PATH` or `EMU_ARCHIVE_STORAGE_PATH` is not set or the volume is not mounted. See [Manage storage and archiving](storage-and-archive.md).
- If port `3399` is already allocated, stop the old app container or choose another host port, for example `3401:3399`.

## Related topics

[Configuration](configuration.md) · [Storage and archive](storage-and-archive.md) · [Docker operations](docker-operations.md) · [Framework update](framework-update.md)
