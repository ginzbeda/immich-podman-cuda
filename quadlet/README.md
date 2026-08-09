# Quadlet Deployment (systemd-managed)

This directory contains [Podman Quadlet](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) units that run the same Immich stack as `docker-compose.yml`, but managed natively by systemd. This gives you:

* **Start on boot** — no manual `podman-compose up` after a reboot.
* **Real dependency handling** — systemd `Requires=`/`After=` instead of compose's fire-and-forget `depends_on`.
* **Automatic image updates** — via `podman-auto-update.timer` (`AutoUpdate=registry`).
* **No `podman-compose` dependency** — pure Podman + systemd.

The compose file remains the portable reference configuration; both consume the same `.env` variables.

## Prerequisites

* Podman **>= 4.4** (5.x recommended)
* [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) with CDI specs generated to a **persistent** path:
  ```bash
  sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
  ```

  > **This matters more for Quadlet than for compose.** `nvidia-ctk` defaults to writing `/var/run/cdi/nvidia.yaml`, which is tmpfs and is wiped on every reboot. With compose you regenerate it by hand before bringing the stack up, so you never notice. These units start automatically at boot, so a spec that only lives in `/run` means `AddDevice=nvidia.com/gpu=all` fails to resolve and both GPU containers refuse to start after a reboot. Write it to `/etc/cdi/`. Verify with `ls -l /etc/cdi/nvidia.yaml`.
* For rootless: allow your user's services to run without an active login session:
  ```bash
  loginctl enable-linger $USER
  ```

## Installation (rootless)

1. **Environment file.** The units read `~/dev/repo/immich-app/.env` directly — the same file compose uses, so there is one source of truth. It serves double duty: passing variables into the containers, and letting systemd expand `${HOST_MEDIA_LOCATION}` / `${HOST_LIBRARY_LOCATION}` / `${HOST_IMPORT_LOCATION}` in the volume mounts.

   If you clone this repo elsewhere, update the `EnvironmentFile=` paths in the `.container` units to match.

2. **Install the units.** Copy the unit files only — not this README, which Quadlet would have to skip:
   ```bash
   mkdir -p ~/.config/containers/systemd
   cp quadlet/*.container quadlet/*.volume quadlet/*.network ~/.config/containers/systemd/
   systemctl --user daemon-reload
   ```

   Check what Quadlet generated before starting anything:
   ```bash
   QUADLET_UNIT_DIRS=$PWD/quadlet /usr/local/libexec/podman/quadlet --user --dryrun
   ```

3. **Start the stack** (redis and postgres start automatically as dependencies):
   ```bash
   systemctl --user start immich-server immich-machine-learning
   ```

4. **Verify:**
   ```bash
   systemctl --user status immich-server
   journalctl --user -u immich-server -f
   ```

Boot autostart is handled by the `[Install] WantedBy=default.target` line in each unit — no `systemctl enable` needed (Quadlet units cannot be enabled manually).

5. **Optional — automatic image updates:**
   ```bash
   systemctl --user enable --now podman-auto-update.timer
   ```

## Migrating from podman-compose

1. Stop the compose stack: `podman-compose down` — **without** `-v`/`--volumes`, which would destroy the Postgres cluster and the model cache.
2. Media and the Redis cache live on host bind mounts (`HOST_MEDIA_LOCATION` / `HOST_LIBRARY_LOCATION` / `HOST_IMPORT_LOCATION`), so they carry over untouched.
3. **Postgres and the ML model cache live in named volumes**, and the `.volume` units deliberately adopt the *existing* compose-created ones via `VolumeName=`:

   | Unit | `VolumeName=` | Why |
   |---|---|---|
   | `immich-pgdata.volume` | `immich_pgdata` | Holds the live cluster. Renaming it silently `initdb`s a blank database. |
   | `immich-model-cache.volume` | `immich_model-cache` | Avoids re-downloading several GB of CUDA models. |

   Nothing needs copying — verify with `podman volume ls | grep immich` before starting.
4. Follow the installation steps above.

## Notes

* **Hostnames:** the units attach network aliases (`redis`, `database`, `immich-machine-learning`) matching the compose service names, so the existing `.env` values (`DB_HOSTNAME=immich_postgres`, default `REDIS_HOSTNAME=redis`) resolve unchanged.
* **Image pinning:** `postgres` and `redis` are pinned by digest to exactly match `docker-compose.yml`. The Postgres image **must** stay on the VectorChord build — `tensorchord/pgvecto-rs` lacks the extension current Immich requires, and pointing it at the existing cluster will fail.
* **Port binding:** `immich-server` publishes on `127.0.0.1:2283`, matching compose. Reach it from Windows through WSL's localhost forwarding; put a reverse proxy in front for LAN access.
* **`AutoUpdate=registry`** is set on `immich-server` and `immich-machine-learning` only. The digest-pinned database and cache are intentionally excluded — an unattended Postgres major-version jump would break the cluster.
* **SELinux:** if enforcing, append `:z` to the bind-mount `Volume=` lines, as with compose.
* **Storage tiers:** `immich-server` mounts four host paths — `${HOST_MEDIA_LOCATION}` at `/usr/src/app/upload` (SSD: thumbs and in-flight uploads), `${HOST_LIBRARY_LOCATION}` over `/usr/src/app/upload/library` (HDD: managed originals), `${HOST_BULK_LOCATION}` over `encoded-video/`, `backups/` and `profile/` (HDD: large or cold Immich output), and `${HOST_IMPORT_LOCATION}` at `/usr/src/app/external` read-only (HDD: External Library source). **Mount order matters** — nested mounts must follow the mount they sit inside, so the three `${HOST_BULK_LOCATION}` lines come after the upload root. Container paths are fixed, so moving the host side of any of these needs no database change. Keep `HOST_MEDIA_LOCATION` pointed at a directory Immich owns exclusively — aim it at a folder that already holds photos and Immich will create `thumbs/`, `upload/`, `encoded-video/` and `backups/` interleaved with your own folders.
* ⚠️ **`encoded-video/` is not a pure cache.** Motion-photo companion videos live there as their only copy, so it must be copied and verified rather than cleared and regenerated.

