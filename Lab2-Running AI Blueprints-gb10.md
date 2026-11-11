### Lab: Deploy NVIDIA Video Search and Summarization (VSS) on DGX Spark 🧪

#### 🔧 Prerequisites

Ensure your environment meets the following:

*   **Hardware/Software**:
    *   DGX Spark device with ARM64 & Blackwell GPU
    *   DGX OS ≥ 7.2.3
    *   NVIDIA driver ≥ 580.82.09 (via `nvidia-smi`)
    *   CUDA ≥ 13.0 (`nvcc --version`)
*   **Docker**:
    *   Installed and running (`docker --version` & `docker compose version`)
*   **NGC Access**:
    *   NVIDIA Container Registry login credentials
*   **Optional for hybrid deployment**:
    *   NVIDIA API Key for remote model endpoints

***

#### 1. Verify Environment

```bash
nvidia-smi | grep "Driver Version"
nvcc --version
docker --version && docker compose version
```

Ensure correct driver, CUDA, and Docker versions.

***

#### 2. Configure Docker

*   Add your user to the `docker` group:
    ```bash
    sudo usermod -aG docker $USER
    newgrp docker
    ```
*   Enable NVIDIA Container Runtime:
    ```bash
    sudo nvidia-ctk runtime configure --runtime=docker
    sudo systemctl restart docker
    ```
*   Test NVIDIA GPU in Docker:
    ```bash
    sudo docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi 
    ```

***

#### 3. Clone the VSS Repository

```bash
git clone https://github.com/NVIDIA-AI-Blueprints/video-search-and-summarization.git
cd video-search-and-summarization
``` 

---

#### 4. Launch Cache Cleaner
In a separate terminal:
```bash
sudo sh deploy/scripts/sys_cache_cleaner.sh
```

*(Add `&` to run in background.)*

***

#### 5. Create Shared Docker Network

```bash
docker network create vss-shared-network
```
---

#### 6. Log into NVIDIA Container Registry
```bash
export NGC_API_KEY="<YOUR_NGC_API_KEY>"
echo "$NGC_API_KEY" | docker login nvcr.io --username '$oauthtoken' --password-stdin
```



### 7. Deploy Fully Local VSS Event Reviewer

We will be deploying the following architecture using AI Blueprints:
---

| Mode             | VLM           | LLM               | Embeddings    | CV Pipeline |
|------------------|---------------|-------------------|---------------|-------------|
| **Event Reviewer** | Local only    | Not used           | N/A           | Local       |

---

1. Enter config directory:
    ```bash
    cd deploy/docker/event_reviewer/
    ```

2.  Add your NGC key and VSS image to `.env`:
    ```bash
    echo "NGC_API_KEY=<YOUR_NGC_API_KEY>" >> .env
    echo "VSS_IMAGE=nvcr.io/nvidia/blueprint/vss-engine-sbsa:2.4.0" >> .env
    ```
3.  Start the main stack:
    ```bash
    IS_SBSA=1 IS_AARCH64=1 ALERT_REVIEW_MEDIA_BASE_DIR=/tmp/alert-media-dir \
      docker compose up
    ```
4.  In a second terminal, enable CV pipeline:
    ```bash
    cd video-search-and-summarization/examples/cv-event-detector
    echo "NV_CV_EVENT_DETECTOR_IMAGE=nvcr.io/nvidia/blueprint/nv-cv-event-detector-sbsa:2.4.0" >> .env
    IS_SBSA=1 IS_AARCH64=1 ALERT_REVIEW_MEDIA_BASE_DIR=/tmp/alert-media-dir \
      docker compose up
    ```
5.  Confirm all containers are `Up` via `docker ps`.
6.  Validate services:
    ```bash
    curl -I http://localhost:7862
    curl -I http://localhost:7860
    ```
    *(For remote Spark, replace `localhost` with Spark’s IP/hostname)*

***

#### 8. Test the Deployment

**Event Reviewer**:

*   Use CV UI (`http://localhost:7862`) to upload videos
*   Check results in Alert Inspector (`http://localhost:7860`)

***

#### 9. Cleanup & Teardown

```bash
# Event Reviewer:
cd deploy/docker/event_reviewer/
IS_SBSA=1 IS_AARCH64=1 ALERT_REVIEW_MEDIA_BASE_DIR=/tmp/alert-media-dir docker compose down

cd ../../examples/cv-event-detector/
IS_SBSA=1 IS_AARCH64=1 ALERT_REVIEW_MEDIA_BASE_DIR=/tmp/alert-media-dir docker compose down
``` 
To remove an existing network:

```bash
docker network rm vss-shared-network
``` 

---
