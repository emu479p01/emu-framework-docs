# Troubleshoot the system

## Audience

Framework administrators and deployment operators.

## Prerequisites

Access to application status, logs, host storage, and Docker commands when applicable.

## App does not start

Check ports, free disk space, container state, and logs:

```sh
docker compose ps
docker compose logs --tail=200 app updater
```

## Update check fails

Confirm access to GitHub Releases and GHCR. A proxy, DNS rule, API rate limit, missing updater token, or private package visibility can block the request.

## Database errors

Stop writes, preserve the database and WAL files, and create a filesystem copy before attempting recovery. Never edit a production SQLite file directly.

Open **Settings → System Maintenance** and inspect `journal_mode`, `foreign_keys`, `busy_timeout`, WAL checkpoint settings, and the integrity result for both databases. Stop all old containers if a lock persists; never run two writer containers against the same volume.

## Related topics

[Recovery](recovery.md) · [Database storage](database-storage.md)
