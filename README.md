# Immich Deployment: NVIDIA-Accelerated Podman Stack

This repository contains a rootless-ready Podman Compose configuration for [Immich](https://immich.app/), optimized for hardware-accelerated machine learning and high-performance tiered storage.

## Key Configurations

### 1. Podman + NVIDIA GPU Passthrough
The stack is configured to leverage CUDA for the `immich-machine-learning` service. 
* **Note:** Unlike Docker, Podman requires the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) to be configured with **CDI (Container Device Interface)** support.
* **Logic:** The `devices` block in the compose file utilizes the `nvidia.com/gpu=all` CDI device for seamless passthrough.

## Storage Architecture: Tiered SSD/HDD Deployment

This stack is engineered for a "Hot/Cold" storage split to maximize performance and organization while maintaining bulk capacity on high-density drives.

### 1. High-Performance "Hot" Tier (SSD)
* **Path:** Defined by `IMMICH_CONFIG_LOCATION`.
* **Logic:** Stores the **PostgreSQL** database and **Redis** cache. 
* **Benefit:** Since facial recognition and metadata searches are IO-intensive, keeping the database on an SSD ensures near-instant UI response times.

### 2. Bulk "Cold" Tier (HDD)
* **Path:** Defined by `IMMICH_STORAGE_LOCATION`.
* **Logic:** Stores all **uploaded media**, generated **thumbnails**, and **transcoding** results.
* **Benefit:** Optimized for large-scale storage. Storing these large files on an HDD provides massive scale at a lower cost without impacting system speed.

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