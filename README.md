# Immich Deployment: NVIDIA-Accelerated Podman Stack

This repository contains a rootless-ready Podman Compose configuration for [Immich](https://immich.app/), optimized for hardware-accelerated machine learning and external media management.

## Key Configurations

### 1. Data Persistence and State
* **Immich Data:** System thumbnails and encoded videos are stored at `IMMICH_DATA_LOCATION`.
* **Database:** PostgreSQL data is persisted via a named volume (`pgdata`) to ensure resilience across pod restarts. We utilize `pgvecto-rs` to support high-performance vector searches for facial recognition.