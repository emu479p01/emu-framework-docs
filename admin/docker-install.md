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

If containers are created through Docker Desktop, create a user-defined network such as `emu-net` and put both containers on it. The app container must have:

```text
Image: ghcr.io/emu479p01/emu-framework:<version>
Port: 3399 -> 3399
Volume: emu-data -> /data
EMU_DEPLOYMENT_MODE=docker
EMU_UPDATER_URL=http://updater:3400
EMU_UPDATER_TOKEN=<same token as updater>
EMU_DB_PATH=/data/data.db
EMU_DESIGNER_DB_PATH=/data/designer.db
```

The updater container must use the matching updater image, the same token, the same `emu-net` network, the `emu-data` volume, and a Docker socket mount. Set its app container name to match the actual app container name. Do not publish updater port `3400` to the host; it is an internal service.

After changing environment variables, recreate the app container. Restarting an existing container does not add new environment variables.

## Existing installation and data migration

Before removing an existing container, inspect its mounts:

```sh
docker inspect <app-container> --format '{{json .Mounts}}'
```

The database files should be mounted at `/data`. A named volume such as `emu-data` can be reused safely when recreating the container. Do not run `docker compose down -v` unless deleting all application data is intentional.

If the existing container uses an anonymous volume, copy it to a named volume before recreating the app:

```sh
docker volume create emu-data
docker run --rm -v <old-volume>:/from -v emu-data:/to alpine sh -c 'cp -a /from/. /to/'
```

Verify that `data.db` and `designer.db` exist in the new volume before removing the old container.

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
