# Windows installation is no longer supported

Starting with v0.5.0.0, EmuFramework production deployment is Docker-only. The Windows host launcher, portable runtime, installer, and host update/restore scripts have been removed.

On a Windows workstation or server, install Docker Desktop or another supported Docker Engine environment and follow [Install with Docker](docker-install.md). Keep `data.db`, `designer.db`, and `.emu-secret.key` in the persistent `/data` volume; do not run an older Windows process against the same databases.

For migration, stop the Windows process, create and verify a Full backup, preserve `.emu-secret.key` separately, and follow the existing-installation steps in the Docker guide.

## Related topics

[Docker installation](docker-install.md) · [Framework update](framework-update.md) · [Restore](restore.md)
