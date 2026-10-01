# Self-Hosted Immich Setup (Raspberry Pi 4B + Offloaded Windows ML)

A hybrid self-hosted Immich deployment designed to keep storage and core server operations running 24/7 on a **Raspberry Pi 4B**, while offloading heavy machine learning compute tasks (Smart Search, Face Detection/Recognition, and OCR) to a dedicated **Windows Laptop with an NVIDIA RTX 3060 GPU**.

---

## Architecture Overview

```text
                        +-----------------------+
                        |    Remote / Mobile    |
                        +-----------+-----------+
                                    |
                                    | Tailscale (Port 2283)
                                    v
+-----------------------------------------------------------------------+
| RASPBERRY PI 4B (Server - 24/7)                                       |
|                                                                       |
|  +-- Docker Stack -------------------------------------------------+  |
|  |  * immich-server (Port 2283)                                  |  |
|  |  * immich-postgres (Metadata & Vector Embeddings)               |  |
|  |  * immich-redis (Job Messaging & Queues)                        |  |
|  +-----------------------------------------------------------------+  |
|                                                                       |
|  +-- Storage ------------------------------------------------------+  |
|  |  * 1 TB External Drive (~915 GiB usable - ext4)                 |  |
|  +-----------------------------------------------------------------+  |
+-----------------------------------+-----------------------------------+
                                    |
                                    | Batch Processing API Requests
                                    | (Tailscale / Local Network)
                                    v
+-----------------------------------------------------------------------+
| WINDOWS LAPTOP (Machine Learning Worker - On Demand)                  |
|                                                                       |
|  * Hardware: NVIDIA GeForce RTX 3060 Mobile GPU                       |
|  * Docker Desktop: immich-machine-learning (CUDA enabled)             |
+-----------------------------------------------------------------------+
````

## Infrastructure & Configuration

### 1. Host Machine (Raspberry Pi 4B)
* **OS / Network:** [Raspberry Pi OS Lite](https://www.raspberrypi.com/software/operating-systems/) connected to a private Tailnet via [**Tailscale**](https://tailscale.com/).
* **Storage Partitioning:**
  * **1 TB Dedicated Media Drive:** Displays as **~915 GiB usable** in Linux due to binary system measurement ($1\text{ TB} \approx 931\text{ GiB}$) and `ext4` filesystem overhead/inodes. A [Samsung 990 EVO Plus SSD](https://www.amazon.com/dp/B0DHLFWBQ1) plugged into a [UGREEN SSD Enclosure](https://www.amazon.com/dp/B09T97Z7DM) with a USB-A 3.0 connection to the Raspberry Pi.
  * **Optimized Reserved Blocks:** Reserved `ext4` blocks reduced from 5% to 1% to reclaim ~35–40 GB of space:
    ```bash
    sudo tune2fs -m 1 /dev/sdX1
    ```
* [**Immich Server Services:**](https://immich.app/) Runs `immich-server`, `postgres`, and `redis` via Docker Compose. The local `immich-machine-learning` container is disabled or unused on the Pi.

### 2. Offloaded Machine Learning Worker (Windows Laptop)
* **GPU Acceleration:** NVIDIA RTX 3060 utilized via Docker Desktop WSL2 backend.
* **Container Configuration:** Set to `restart: always` and configured to run on login.
* **Network Binding:** Listens on port `3003` to receive job payloads forwarded by the Pi server.

---

## Key System & Machine Learning Settings

Configured under **Administration** → **Settings** → **Machine Learning Settings** in the Immich Web UI:

* **Facial Recognition:**
  * **Max Recognition Distance:** Recommended `0.3` – `0.7` (Default tuned for accurate cluster identification without combining similar relatives).
  * **Min Detection Score:** `0.7` (Prevents false face detections on background textures).
  * **Min Recognized Faces:** `3` (Keeps one-off background strangers out of the primary People tab).
* **Concurrency:**
  * **OCR Concurrency:** Increased to **`3`–`4`** jobs to maximize RTX 3060 GPU throughput over network calls.

---

## Lifecycle of Offline / On-Demand ML Processing

Because the laptop is not running 24/7, Immich handles job scheduling dynamically:

1. **New Media Ingestion:** Photos/videos uploaded via `immich-go` or mobile apps are stored on the Pi's drive immediately. Thumbnails and EXIF metadata generate in real time.
2. **Offline Backoff:** If the laptop is powered off, the Pi attempts to reach the ML endpoint, fails, and places pending jobs into an exponential retry backoff queue in Redis.
3. **Triggering Processing on Laptop Boot:**
   * Boot the Windows laptop and launch Docker Desktop.
   * Go to **Administration** → **Jobs** on the Immich Web UI.
   * Click **Resume** (if paused) and select **"Missing"** under **Smart Search**, **Face Detection**, and **OCR** to force an immediate database audit and start GPU processing.

---

## Maintenance & Operations Guide

### Mass Ingestion & Bulk Imports ([`immich-go`](https://github.com/simulot/immich-go))

To avoid web UI timeouts and ensure metadata integrity during initial bulk uploads (e.g., Google Takeout, iCloud, or large local photo directories), use **`immich-go`**.

#### Key Advantages for this Setup
* **Google Photos Takeout Matching:** Automatically parses `.json` metadata sidecars, restoring correct creation dates, GPS tags, descriptions, and album structures.
* **Resource Optimization:** Reduces CPU/RAM overhead on the Raspberry Pi during mass ingestion compared to web uploads.
* **Deduplication:** Prevents re-uploading duplicate assets if a migration job is interrupted.

---

### Common Usage Commands

Run these commands from your laptop terminal where the photo archives reside:

#### 1. Google Photos Takeout Migration
```powershell
.\immich-go upload from-google-photos `
  --server="http://<PI-TAILSCALE-IP>:2283" `
  --api-key="YOUR_IMMICH_API_KEY" `
  path/to/takeout-*.zip

### Updating Immich

#### 1. Server Update (Raspberry Pi)
```bash
cd ~/immich-app
docker compose pull
docker compose up -d
docker image prune -f  # Reclaims space on Pi drive
```
#### 2. Worker Update (Windows Laptop)
```bash
docker compose pull
docker compose up -d
docker image prune -f  # Deletes multi-gigabyte stale ML images (preserves model-cache volume)
```
#### 3. Mobile Client

Update the mobile app via Google Play / App Store immediately following a server update to ensure protocol version alignment.

#### Power Outage / Dirty Shutdown Recovery

If the Raspberry Pi loses power unexpectedly:
1. Power on the Pi and SSH into the system.
2. Verify container health: docker compose ps.
3. If containers are stopped, bring the stack up: docker compose up -d. PostgreSQL automatically replays WAL logs to ensure database integrity.
4. Verify storage mount point with df -h.
