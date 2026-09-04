# Quadlet Deployment (systemd-managed)

This directory contains [Podman Quadlet](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) units that run the same Immich stack as `docker-compose.yml`, but managed natively by systemd. This gives you:

* **Start on boot** — no manual `podman-compose up` after a reboot.
* **One handle for the whole stack** — the four services run in a pod (`immich.pod`), and Quadlet gives every member `BindsTo=immich-pod.service`. Stopping the pod unit stops all of them; there is no "I stopped the server but postgres is still up" state.
* **Automatic image updates** — via `podman-auto-update.timer` (`AutoUpdate=registry`).
* **No `podman-compose` dependency** — pure Podman + systemd.

The compose file remains the portable reference configuration; both consume the same `.env` variables.

Day-to-day you should not need any of the commands below — use [`scripts/immichctl`](../scripts/immichctl):

```bash
./scripts/immichctl install    # copy units + daemon-reload + validate
./scripts/immichctl start      # or: stop | restart | status | logs | ps | update
```

## Why a pod (and what it costs)

Pod members share one network namespace, so Podman writes `/etc/hosts` entries for each member pointing at `127.0.0.1`, keyed on **`ContainerName`**. Per-container `NetworkAlias=` no longer applies — the alias belongs to the pod's netns — so two Immich defaults stop resolving:

| Setting | Value it wants | `ContainerName` | Resolves? |
|---|---|---|---|
| `DB_HOSTNAME` | `immich_postgres` | `immich_postgres` | yes |
| `REDIS_HOSTNAME` | `redis` | `immich_redis` | **no** |
| `IMMICH_MACHINE_LEARNING_URL` | `http://immich-machine-learning:3003` | `immich_machine_learning` | **no** |

Editing `.env` to fix those would break the single-source-of-truth it shares with `docker-compose.yml`, which still needs service-name hostnames. So the divergence is isolated in **`immich-pod.env`**, layered after `.env` in each unit's `EnvironmentFile=` list (systemd applies them in order; last wins). `.env` is untouched and compose keeps working.

Port publishing also moves: Podman only accepts `--publish` at pod-creation time, so `PublishPort=127.0.0.1:2283:2283` lives on `immich.pod`, not on `immich-server.container`. No collisions inside the shared netns — server 2283, ML 3003, postgres 5432, redis 6379.

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

1. **Environment file.** The units read `~/dev/repo/immich-app/.env` directly — the same file compose uses, so there is one source of truth. It serves double duty: passing variables into the containers, and letting systemd expand `${HOST_MEDIA_LOCATION}` / `${HOST_LIBRARY_LOCATION}` in the volume mounts.

   If you clone this repo elsewhere, update the `EnvironmentFile=` paths in the `.container` units to match.

2. **Install the units.** `./scripts/immichctl install` does this and validates the result. By hand it is:
   ```bash
   mkdir -p ~/.config/containers/systemd
   cp quadlet/*.container quadlet/*.volume quadlet/*.network quadlet/*.pod ~/.config/containers/systemd/
   systemctl --user daemon-reload
   ```

   Copy the unit files only — a bare `cp quadlet/*` drops this README and `immich-pod.env` into the unit directory. (`immich-pod.env` is read from the repo via an absolute `EnvironmentFile=` path, so it must *not* be installed.)

   Check what Quadlet generated before starting anything:
   ```bash
   QUADLET_UNIT_DIRS=$PWD/quadlet /usr/local/libexec/podman/quadlet --user --dryrun
   ```

3. **Start the stack** — one unit brings up all four containers plus the pod infra container:
   ```bash
   systemctl --user start immich-pod.service
   ```

4. **Verify:**
   ```bash
   ./scripts/immichctl status
   curl -sf http://127.0.0.1:2283/api/server/ping    # {"res":"pong"}
   ```

Boot autostart is handled by the `[Install] WantedBy=default.target` line in each unit — no `systemctl enable` needed (Quadlet units cannot be enabled manually).

## Managing the stack

| Task | `immichctl` | Underlying command |
|---|---|---|
| Start everything | `start` | `systemctl --user start immich-pod.service` |
| Stop everything | `stop` | `systemctl --user stop immich-pod.service` |
| Restart | `restart` | stop, then start |
| Health + unit states | `status` | — |
| Follow logs | `logs [server\|ml\|db\|redis]` | `journalctl --user -u immich-server -f` |
| List containers | `ps` | `podman ps --filter pod=immich` |
| Update images | `update` | `podman auto-update` |
| Tear down containers | `remove` | `podman pod rm -f immich` |
| Remove installed units | `uninstall` | — |

**Never `podman stop immich_server`.** `Restart=always` makes systemd bring it straight back. Go through `immichctl` or `systemctl --user`.

`remove` and `uninstall` deliberately leave `immich_pgdata` and `immich_model-cache` alone — the database is the only copy of your albums, faces and asset metadata. Deleting it is a separate, explicit `podman volume rm immich_pgdata`.

Restarting a single service still works and does not disturb the rest of the pod:
```bash
systemctl --user restart immich-server
```

5. **Optional — automatic image updates:**
   ```bash
   systemctl --user enable --now podman-auto-update.timer
   ```

## Migrating from podman-compose

1. Stop the compose stack: `podman-compose down` — **without** `-v`/`--volumes`, which would destroy the Postgres cluster and the model cache.
2. Media and the Redis cache live on host bind mounts (`HOST_MEDIA_LOCATION` / `HOST_LIBRARY_LOCATION`), so they carry over untouched.
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
* **Storage tiers:** `immich-server` mounts two host paths — `${HOST_MEDIA_LOCATION}` at `/usr/src/app/upload` (SSD: Immich's working dirs — thumbs, transcodes, profile, backups) and `${HOST_LIBRARY_LOCATION}` at `/usr/src/app/external` **read-only** (HDD: the curated originals, indexed in place as an External Library). Container paths are fixed, so moving the host side of either needs no database change. Keep `HOST_MEDIA_LOCATION` pointed at a directory Immich owns exclusively — aim it at a folder that already holds photos and Immich will create `thumbs/`, `upload/`, `encoded-video/` and `backups/` interleaved with your own folders. And keep the curated tree on `:ro` at `/usr/src/app/external`: mounted at `/usr/src/app/upload/library` instead it becomes the read-write *managed* library, which the Storage Template Migration job will rename and reshuffle.

