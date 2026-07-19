# Quadlet Deployment (systemd-managed)

This directory contains [Podman Quadlet](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) units that run the same Immich stack as `docker-compose.yml`, but managed natively by systemd. This gives you:

* **Start on boot** — no manual `podman-compose up` after a reboot.
* **Real dependency handling** — systemd `Requires=`/`After=` instead of compose's fire-and-forget `depends_on`.
* **Automatic image updates** — via `podman-auto-update.timer` (`AutoUpdate=registry`).
* **No `podman-compose` dependency** — pure Podman + systemd.

The compose file remains the portable reference configuration; both consume the same `.env` variables.

## Prerequisites

* Podman **>= 4.4** (5.x recommended)
* [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) with CDI specs generated:
  ```bash
  sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
  ```
* For rootless: allow your user's services to run without an active login session:
  ```bash
  loginctl enable-linger $USER
  ```

## Installation (rootless)

1. **Stage the environment file.** The units read `~/.config/immich/immich.env` — both to pass variables into the containers and so systemd can expand `${IMMICH_STORAGE_LOCATION}` / `${IMMICH_CONFIG_LOCATION}` in the volume mounts:
   ```bash
   mkdir -p ~/.config/immich
   cp .env ~/.config/immich/immich.env
   ```

2. **Install the units:**
   ```bash
   mkdir -p ~/.config/containers/systemd
   cp quadlet/* ~/.config/containers/systemd/
   systemctl --user daemon-reload
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

1. Stop the compose stack: `podman-compose down`
2. Media, Postgres, and Redis data live on host bind mounts (`IMMICH_STORAGE_LOCATION` / `IMMICH_CONFIG_LOCATION`), so they carry over untouched.
3. The ML model cache uses a fresh named volume (`immich-model-cache`); models re-download automatically on first use. To skip the re-download:
   ```bash
   podman volume create immich-model-cache
   podman run --rm -v immich_model-cache:/from -v immich-model-cache:/to alpine cp -a /from/. /to/
   ```
4. Follow the installation steps above.

## Notes

* **Hostnames:** the units attach network aliases (`redis`, `database`, `immich-machine-learning`) matching the compose service names, so the existing `.env` values (`DB_HOSTNAME=immich_postgres`, default `REDIS_HOSTNAME=redis`) resolve unchanged.
* **Image tags** are pinned in the `.container` files (`release` / `release-cuda`) rather than read from `IMMICH_VERSION`; edit the `Image=` lines to pin a specific version.
* **SELinux:** if enforcing, append `:z` to the bind-mount `Volume=` lines, as with compose.
