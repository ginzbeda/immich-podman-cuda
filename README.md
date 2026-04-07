# Immich Deployment: NVIDIA-Accelerated Podman Stack

This repository contains a rootless-ready Podman Compose configuration for [Immich](https://immich.app/), optimized for hardware-accelerated machine learning and high-performance tiered storage.

## Key Configurations

### 1. Podman + NVIDIA GPU Passthrough
The stack is configured to leverage CUDA for the `immich-machine-learning` service. 
* **Note:** Unlike Docker, Podman requires the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) to be configured with **CDI (Container Device Interface)** support.
* **Optimization:** This setup explicitly uses the `immich-machine-learning:release-cuda` image tag. This ensures that the container includes the necessary CUDA libraries to utilize the hardware without manual driver injection.

## Storage Architecture: Tiered SSD/HDD Deployment

This stack is engineered for a "Hot/Cold" storage split to maximize performance and organization while maintaining bulk capacity on high-density drives.

### 1. High-Performance "Hot" Tier (SSD)
* **Path:** Defined by `IMMICH_CONFIG_LOCATION`.
* **Logic:** Stores the **PostgreSQL** database and **Redis** cache. 
* **Benefit:** Since facial recognition and metadata searches are IO-intensive, keeping the database on an SSD ensures near-instant UI response times and snappy "Topic" filtering.

### 2. Bulk "Cold" Tier (HDD)
* **Path:** Defined by `IMMICH_STORAGE_LOCATION`.
* **Logic:** Stores all **uploaded media**, generated **thumbnails**, and **transcoding** results.
* **Benefit:** Optimized for large-scale storage. By using a **Topic-Based Storage Template**, the physical folder structure on the HDD mirrors your personal organization (e.g., `/Brazil/2026/04/...`).

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

* **Topic-Based Storage Template:** To maintain a human-readable library that matches your personal organization, enable the **Storage Template** in Immich Settings and set it to: `{{album}}/{{y}}/{{MM}}/{{filename}}`.
* **SELinux/Permissions:** For rootless environments, ensure your mount points have the correct user namespace permissions. If SELinux is enforcing, append `:z` or `:Z` to your volume mounts to handle relabeling automatically.
* **ML Model TTL:** This stack is configured with `IMMICH_MACHINE_LEARNING_MODEL_TTL=300`. This keeps the ML models in GPU memory for 5 minutes, balancing processing speed with VRAM efficiency.