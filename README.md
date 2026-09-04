# Immich Deployment: Hardware-Accelerated Podman Stack

This repository contains a rootless-ready Podman Compose configuration for Immich, optimized for hardware-accelerated machine learning and high-performance tiered storage (SSD/HDD split).

## 1. Storage Architecture: Tiered Deployment

This stack is engineered for a "Hot/Cold" storage split to maximize UI performance while maintaining bulk capacity on high-density drives. Two host paths are mounted into `immich-server`:

| Variable | Container path | Tier | Holds |
|---|---|---|---|
| `HOST_MEDIA_LOCATION` | `/usr/src/app/upload` | SSD | Immich's working dirs — thumbnails, transcodes, profile images, in-flight uploads, DB backups |
| `HOST_LIBRARY_LOCATION` | `/usr/src/app/external` (**ro**) | HDD | Your curated originals (`Photos/`, `Mems/`, `videos/`, `Edits/`), indexed in place and never written to |

The originals are an **External Library**: Immich reads them where they sit and your folder structure is authoritative. Nothing is copied into a managed library, so `/usr/src/app/upload/library` stays empty by design.

> **The `:ro` and the `/usr/src/app/external` target are both load-bearing.** Mounting the curated tree at `/usr/src/app/upload/library` instead would make it Immich's *managed* library — read-write, and subject to the Storage Template Migration job, which would rename and move your originals into Immich's own date-based layout.

### High-Performance Tier (SSD)
* **Usage:** Thumbnails, transcodes, profile images, Redis cache. The PostgreSQL cluster lives in the `immich_pgdata` podman named volume rather than a bind mount, which avoids NTFS/DrvFs permission problems under WSL2.
* **Logic:** IO-intensive operations (metadata, facial recognition, thumbnail reads) stay on fast storage for a responsive UI.

### Bulk Media Tier (HDD)
* **Usage:** Original high-resolution photos and videos.
* **Logic:** Optimized for large-scale storage capacity.

> **Give `HOST_MEDIA_LOCATION` a directory Immich owns exclusively.** Point it at a folder that already contains your photos and Immich will create its working directories — `thumbs/`, `upload/`, `encoded-video/`, `backups/`, `library/` — interleaved with your own. Existing media belongs on the `HOST_LIBRARY_LOCATION` side, indexed in place as an External Library.

---

## 2. Deployment Steps

### Step 1: Prepare Environment
Copy `.env.example` to `.env` and populate it with your local paths and credentials:

```bash
cp .env.example .env
chmod 600 .env    # it holds DB_PASSWORD
```

Example paths for Linux/WSL2:

```
HOST_MEDIA_LOCATION=/mnt/e/Media/immich-data
HOST_LIBRARY_LOCATION=/mnt/d/Media/library
DB_PASSWORD=your_secure_password
```

### Step 2: Verify GPU Visibility
If utilizing NVIDIA hardware, ensure the NVIDIA Container Toolkit is configured with CDI support, writing the spec to a **persistent** path:

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
podman run --rm --device nvidia.com/gpu=all ubuntu nvidia-smi
```

`nvidia-ctk` defaults to `/var/run/cdi/`, which is tmpfs and does not survive a reboot. That is tolerable when you start the stack by hand, but it breaks the Quadlet units below, which start at boot.

### Step 3: Launch
Deploy the stack using podman-compose:
>`podman-compose up -d`

### Alternative: systemd-native deployment (Quadlet pod)

For a production-style setup that starts on boot and is supervised by systemd — no `podman-compose` required — use the Quadlet units in [`quadlet/`](quadlet/README.md). The four services run as a podman **pod**, so the whole stack has a single start/stop handle:

```bash
./scripts/immichctl install     # copy units to ~/.config/containers/systemd, validate
./scripts/immichctl start       # bring up the pod
./scripts/immichctl status      # unit states, containers, volumes, API health
./scripts/immichctl stop        # take the whole stack down
```

Run `./scripts/immichctl --help` for the rest (`restart`, `logs`, `ps`, `update`, `remove`, `uninstall`).

Both deployment methods share the same `.env` file and the same named volumes; the compose file remains the portable reference configuration. Pod-specific hostname overrides live in `quadlet/immich-pod.env` so `.env` stays common to both — see [`quadlet/README.md`](quadlet/README.md) for why.

---

## 3. Data Migration and Organization

### Consolidating Existing Media
New media goes into the curated tree under `HOST_LIBRARY_LOCATION` — never inside `HOST_MEDIA_LOCATION`:

1. **Copy data in.** Add the files under an existing top-level folder (`Photos/`, `Mems/`, `videos/`, `Edits/`) or create a new one. On Windows, Robocopy is recommended for speed:
   ```
   robocopy "D:\Path\To\Source" "D:\Media\library\Source" /E /MT:32 /R:3 /W:5
   ```
2. **Scan.** Administration > Libraries > the external library (import path `/usr/src/app/external`) > **Scan**. Immich indexes the files where they are and leaves your folder structure untouched.
3. **Verify before deleting anything.** Confirm the new asset count and that thumbnails generated, and verify the destination copy by checksum, before removing the source.

> **Do not use managed upload against this layout.** `immich-go` and the Immich CLI copy into the *managed* library at `/usr/src/app/upload/library`, which here resolves to the SSD working dir, not the curated tree — and those assets are then subject to the Storage Template. Keep a single ingest path: files on disk, indexed in place.

---

## 4. Recommended Post-Install Configuration

### Storage Template (Human-Readable Folders)

> **Not applicable to this deployment.** The Storage Template only governs *managed* assets under `/usr/src/app/upload/library`. Here every asset is external, so the template does nothing — your own folder tree already is the human-readable layout. Leave it disabled; the guidance below applies only if you later add managed uploads.

To maintain an organized library that is easy to navigate outside of the Immich UI, enable the **Storage Template** in Settings.

**Recommended Template String:**
>`{{#if album}}{{{album}}}{{else}}{{y}}{{/if}}/{{y}}-{{MM}}-{{dd}}_{{filename}}`

* **Logic:** Files added to an Album are sorted into Event/Topic folders. Files without an album fall back to a Year-based structure. Filenames are prefixed with ISO dates for tactical sorting.

### Manual Mount Verification
To verify that your tiered storage is correctly mapping to the intended drives:

```bash
podman exec immich_server df -h /usr/src/app/upload /usr/src/app/external
podman exec immich_server touch /usr/src/app/external/.wtest   # must fail: read-only
```

The upload root should report your SSD's capacity and `/usr/src/app/external` your bulk drive's. The `touch` **must** fail with "Read-only file system" — if it succeeds, the `:ro` flag was lost and Immich can write into your originals.

### Maintenance & Jobs
After enabling the Storage Template, go to **Administration > Jobs** and run:
1. **Storage Template Migration:** Moves files into the new directory structure.
2. **Duplicate Detection:** Clears redundant data from the ingest process.

Note that Storage Template Migration only relocates *managed* assets. External Library assets stay where they are by design.
