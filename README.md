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