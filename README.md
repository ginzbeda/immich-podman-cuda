# Immich Deployment: Hardware-Accelerated Podman Stack

This repository contains a rootless-ready Podman Compose configuration for Immich, optimized for hardware-accelerated machine learning and high-performance tiered storage (SSD/HDD split).

## 1. Storage Architecture: Tiered Deployment

This stack is engineered for a "Hot/Cold" storage split to maximize UI performance while maintaining bulk capacity on high-density drives. Three host paths are mounted into `immich-server`:

| Variable | Container path | Tier | Holds |
|---|---|---|---|
| `HOST_MEDIA_LOCATION` | `/usr/src/app/upload` | SSD | Thumbnails, transcodes, profile images, in-flight uploads, DB backups |
| `HOST_LIBRARY_LOCATION` | `/usr/src/app/upload/library` | HDD | Managed originals — mounted *over* the upload root's `library/` subdirectory |
| `HOST_IMPORT_LOCATION` | `/usr/src/app/external` (ro) | HDD | External Library source tree, indexed in place and never written to |

### High-Performance Tier (SSD)
* **Usage:** Thumbnails, transcodes, profile images, Redis cache. The PostgreSQL cluster lives in the `immich_pgdata` podman named volume rather than a bind mount, which avoids NTFS/DrvFs permission problems under WSL2.
* **Logic:** IO-intensive operations (metadata, facial recognition, thumbnail reads) stay on fast storage for a responsive UI.

### Bulk Media Tier (HDD)
* **Usage:** Original high-resolution photos and videos.
* **Logic:** Optimized for large-scale storage capacity.

> **Give `HOST_MEDIA_LOCATION` a directory Immich owns exclusively.** Point it at a folder that already contains your photos and Immich will create its working directories — `thumbs/`, `upload/`, `encoded-video/`, `backups/`, `library/` — interleaved with your own. If you want existing media indexed without being moved, that is what `HOST_IMPORT_LOCATION` and an External Library are for.

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
HOST_IMPORT_LOCATION=/mnt/d/Media/immich-import
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

### Alternative: systemd-native deployment (Quadlet)

For a production-style setup that starts on boot and is supervised by systemd — no `podman-compose` required — use the Quadlet units in [`quadlet/`](quadlet/README.md). Both deployment methods share the same `.env` file and the same named volumes; the compose file remains the portable reference configuration.

---

## 3. Data Migration and Organization

### Consolidating Existing Media
If you have existing media libraries, stage them under `HOST_IMPORT_LOCATION` — never inside `HOST_MEDIA_LOCATION`:

1. **Staging:** Create the import tree on your bulk drive, e.g. `D:\Media\immich-import\`.
2. **Copy Data:** Use a robust copy tool to stage your files. On Windows, Robocopy is recommended for speed:
   ```
   robocopy "D:\Path\To\Source" "D:\Media\immich-import\Source" /E /MT:32 /R:3 /W:5
   ```
3. **Ingest — pick one:**
   * **External Library (no copying).** Administration > Libraries > add an external library with import path `/usr/src/app/external`, then **Scan**. Immich indexes the files where they are and leaves your folder structure untouched.
   * **Managed upload.** Use [`immich-go`](https://github.com/simulot/immich-go) with `--folder-as-album` so each source folder becomes an album, or the [Immich CLI](https://immich.app/docs/features/command-line-interface). Immich copies the files into `HOST_LIBRARY_LOCATION` under the Storage Template.
4. **Cleanup:** Only after verifying counts and that thumbnails have generated, the staging folder can be removed.

---

## 4. Recommended Post-Install Configuration

### Storage Template (Human-Readable Folders)
To maintain an organized library that is easy to navigate outside of the Immich UI, enable the **Storage Template** in Settings.

**Recommended Template String:**
>`{{#if album}}{{{album}}}{{else}}{{y}}{{/if}}/{{y}}-{{MM}}-{{dd}}_{{filename}}`

* **Logic:** Files added to an Album are sorted into Event/Topic folders. Files without an album fall back to a Year-based structure. Filenames are prefixed with ISO dates for tactical sorting.

### Manual Mount Verification
To verify that your tiered storage is correctly mapping to the intended drives:

```bash
podman exec immich_server df -h /usr/src/app/upload /usr/src/app/upload/library /usr/src/app/external
```

The upload root should report your SSD's capacity, and both HDD paths your bulk drive's. If `library` reports the same device as the upload root, the override mount is not in effect.

### Maintenance & Jobs
After enabling the Storage Template, go to **Administration > Jobs** and run:
1. **Storage Template Migration:** Moves files into the new directory structure.
2. **Duplicate Detection:** Clears redundant data from the ingest process.

Note that Storage Template Migration only relocates *managed* assets. External Library assets stay where they are by design.
