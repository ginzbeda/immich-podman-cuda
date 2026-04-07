# Immich Deployment: NVIDIA-Accelerated Podman Stack

This repository contains a rootless-ready Podman Compose configuration for [Immich](https://immich.app/), optimized for hardware-accelerated machine learning and external media management.

## Key Configurations

### 1. Podman + NVIDIA GPU Passthrough
The stack is configured to leverage CUDA for the `immich-machine-learning` service. 
* **Note:** Unlike Docker, Podman requires the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) to be configured with **CDI (Container Device Interface)** support.
* **Logic:** The `devices` block in the compose file utilizes the `nvidia.com/gpu=all` CDI device for seamless passthrough.

### 2. Data Persistence and State
* **Immich Data:** System thumbnails and encoded videos are stored at `IMMICH_DATA_LOCATION`.
* **Database:** PostgreSQL data is persisted via a named volume (`pgdata`) to ensure resilience across pod restarts. We utilize `pgvecto-rs` to support high-performance vector searches for facial recognition.

### 3. "Read-Only" External Libraries
To maintain the integrity of an existing photo collection, this setup utilizes the `EXTERNAL_LIBRARY_LOCATION` variable.
* The source media is mounted as `:ro` (Read-Only). 
* This allows Immich to index and generate thumbnails for your collection without risking accidental modification or deletion of the source files.

---

## Deployment Steps

1.  **Prepare Environment**
    Copy the example environment file and populate it with your local paths and credentials:
    ```bash
    cp .env.example .env
    ```

2.  **Verify GPU Visibility**
    Ensure the NVIDIA CDI hooks are working correctly within your Podman environment:
    ```bash
    podman run --rm --device [nvidia.com/gpu=all](https://nvidia.com/gpu=all) ubuntu nvidia-smi
    ```

3.  **Launch**
    Deploy the stack using podman-compose:
    ```bash
    podman-compose up -d
    ```

## Recommended Post-Install Config

* **SELinux/Permissions:** If running in a rootless environment, ensure your mount points (like `IMMICH_DATA_LOCATION`) have the correct user namespace permissions. If SELinux is enforcing, append `:z` or `:Z` to your volume mounts to handle relabeling automatically.
* **ML Model TTL:** This stack is configured with `IMMICH_MACHINE_LEARNING_MODEL_TTL=300`. This keeps the ML models in GPU memory for 5 minutes of inactivity, striking a balance between instant processing response and freeing up VRAM for other system tasks.