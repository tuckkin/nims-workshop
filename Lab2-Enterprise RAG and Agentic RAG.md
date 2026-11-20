# Lab 2 - Deployment Enterprise RAG and Agentic RAG

In this Hands-On Lab you deploy the NVIDIA RAG Blueprint with Docker Compose for a single node deployment, and using NVIDIA-hosted LLM model on cloud.

## 2.1 Register for NVIDIA account and request for API Key

1. Go to https://build.nvidia.com/ and login using your NVIDIA account.

2. Create an API Key
    Refer to https://docs.nvidia.com/nim/large-language-models/latest/getting-started.html#launch-nvidia-nim-for-llms

3. Using the API Key, perform docker login to nvcr.io
    ```bash
    export NGC_API_KEY="<YOUR_NGC_API_KEY>"
    echo "$NGC_API_KEY" | docker login nvcr.io --username '$oauthtoken' --password-stdin
    ```

## 2.2 Access to the local RAG Blueprint Repository

The RAG Blueprint was downloaded from `https://github.com/NVIDIA-AI-Blueprints/rag.git` to the local folder:

1. Access to the cloned copy of the RAG Blueprint:  

   ```bash
   cd ~/workspace/rag
   ``` 

## 2.3 Start Enterprise RAG services

Use the following procedure to start all containers needed for this blueprint.

1. Create a directory to cache the models and export the path to the cache as an environment variable.

   ```bash
   mkdir -p ~/.cache/model-cache
   export MODEL_DIRECTORY=~/.cache/model-cache
   ```

2. Export all the required environment variables to use on-prem and hosted models.

   ```bash
   source deploy/compose/.env
   ```

4. Start all required NIMs by running the following code.

   ```bash
   USERID=$(id -u) docker compose -f deploy/compose/nims.yaml up -d
   ```

5. Check the status of the deployment by running the following code. Wait until all services are up and the `nemotron-ranking-ms` and `nemotron-vlm-embedding-ms`, and `nim-llm-ms` NIMs are in healthy state before proceeding further.

(<b>Note</b>: Most of the RAG components, including the Ranking and Embedding models will be running on-premise, except for LLM model `nim-llm-ms` which is the NVIDIA-hosted LLM model on cloud due to GPU resource constraint)

     ```bash
     watch -n 2 'docker ps --format "table {{.Names}}\t{{.Status}}"'
     ```
    Your output should look similar to the following.

     ```output
        NAMES                                   STATUS

        nemotron-ranking-ms      Up 4 minutes (healthy)
        compose-graphic-elements-1    Up 4 minutes
        compose-page-elements-1       Up 4 minutes
        nemotron-vlm-embedding-ms Up 4 minutes (healthy)
        compose-nemotron-ocr-1   Up 4 minutes
        compose-table-structure-1     Up 4 minutes
     ```


7. Persistent data (Elasticsearch, SeaweedFS, Milvus/etcd, LanceDB, and the ingestor-server scratch dir) is stored in dedicated `rag-vol-*` Docker named volumes, which Docker Compose creates automatically the first time you bring the stack up. For inspection, backup, and reset commands — and for migrating data from the legacy `deploy/compose/volumes/` host directory — see [Manage Persistent Data Volumes](troubleshooting.md#manage-persistent-data-volumes) in the troubleshooting guide.

8. Start the vector db containers from the repo root.

   ```bash
   docker compose -f deploy/compose/vectordb.yaml up -d
   ```


9. Start the ingestion containers from the repo root. This pulls the prebuilt containers from NGC and deploys them on your system.

   ```bash
   docker compose -f deploy/compose/docker-compose-ingestor-server.yaml up -d
   ```

   You can check the status of the ingestor-server and running the following code.

   ```bash
   curl -X 'GET' 'http://workstation_ip:8082/v1/health?check_dependencies=true' -H 'accept: application/json'
   ```

    You should see output similar to the following.

    ```bash
    {
        "message": "Service is up.",
        "databases": [
            ...
        ],
        "object_storage": [
            ...
        ],
        "nim": [
            {
                "service": "Embeddings",
                "status": "healthy",
                ...
            },
            {
                "service": "Summary LLM",
                "status": "healthy",
                ...
            }
        ],
        "processing": [
            {
                "service": "NeMo Retriever Library",
                "status": "healthy",
                ...
            }
        ],
        "task_management": [
            {
                "service": "Redis",
                "status": "healthy",
                ...
            }
        ]
    }
    ```


10. Start the RAG containers from the repo root. This pulls the prebuilt containers from NGC and deploys them on your system.

    ```bash
    docker compose -f deploy/compose/docker-compose-rag-server.yaml up -d
    ```

    You can check the status of the rag-server by running the following code.

    ```bash
    curl -X 'GET' 'http://workstation_ip:8081/v1/health?check_dependencies=true' -H 'accept: application/json'
    ```

    You should see output similar to the following.

    ```bash
    {
        "message": "Service is up.",
        "databases": [
            ...
        ],
        "object_storage": [
            ...
        ],
        "nim": [
        {
            "service": "LLM",
            "status": "healthy",
            ...
        },
        {
            "service": "Embeddings",
            "status": "healthy",
            ...
        },
        {
            "service": "Ranking",
            "status": "healthy",
            ...
        }
      ]
    }
    ```


11. Check the status of the deployment by running the following code.

    ```bash
    docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}"
    ```

    You should see output similar to the following. Confirm all the following containers are running.

    ```output
    CONTAINER ID   NAMES                            STATUS
    88181d20ba30   rag-frontend                     Up 2 minutes
    5cf93ea91d4e   rag-server                       Up 2 minutes
    03ff43bd4f53   compose-nv-ingest-ms-runtime-1   Up 2 minutes (healthy)
    fcc703631b71   ingestor-server                  Up 2 minutes
    77f64a4a5146   compose-redis-1                  Up 2 minutes
    902445432dde   elasticsearch                    Up 3 minutes (healthy)
    340bc8210a0d   seaweedfs                        Up 3 minutes (healthy)
    62eabf1d9f65   nim-llm-ms                       Up 10 minutes (healthy)
    fe2751bfa734   nemotron-ranking-ms         Up 10 minutes (healthy)
    7b5ddabf8be7   compose-graphic-elements-1       Up 10 minutes
    ecfaa5190302   compose-page-elements-1          Up 10 minutes
    ea8c7fdf20d1   nemotron-vlm-embedding-ms   Up 10 minutes (healthy)
    6d62008a9b42   compose-nemotron-ocr-1      Up 10 minutes
    969b9f5c987c   compose-table-structure-1        Up 10 minutes
    ```



## Experiment with the Web User Interface

After the RAG Blueprint is deployed, you can use the RAG UI to start experimenting with it.

1. Open a web browser and access the RAG UI. You can start experimenting by uploading docs and asking questions. For details, see [User Interface for NVIDIA RAG Blueprint](user-interface.md).



## Experiment with the Ingestion API Usage Notebook

After the RAG Blueprint is deployed, you can use the Ingestion API Usage notebook to start experimenting with it. For details, refer to [Experiment with the Ingestion API Usage Notebook](notebooks.md#experiment-with-the-ingestion-api-usage-notebook).



## Shut down services

1. To stop all running services, run the following code.

    ```bash
    docker compose -f deploy/compose/docker-compose-ingestor-server.yaml down
    docker compose -f deploy/compose/nims.yaml down
    docker compose -f deploy/compose/docker-compose-rag-server.yaml down
    docker compose -f deploy/compose/vectordb.yaml down
    ```


## Service Port and GPU Reference

For a complete reference of all services, their port mappings, and GPU assignments, see [Service Port and GPU Reference](service-port-gpu-reference.md)


## Advanced Deployment Considerations

After the first time you deploy the RAG Blueprint successfully, you can consider the following advanced deployment options:

- For information about advanced settings, see [Best Practices for Common Settings](accuracy_perf.md).

- To turn on recommended configurations for accuracy optimized profile set additional configs by running the following code:

   ```bash
   source deploy/compose/accuracy_profile.env
   ```

- To turn on recommended configurations for performance optimized profile set additional configs by running the following code:

   ```bash
   source deploy/compose/perf_profile.env
   ```

- To start just the services specific to RAG or ingestion add the `--profile rag` or `--profile ingest` flag to the code. For example:

   ```bash
   USERID=$(id -u) docker compose -f deploy/compose/nims.yaml --profile rag up -d
   ```

- If you make code changes and want to redeploy services, add the --build flag to your code.  For example:

   ```bash
   docker compose -f deploy/compose/docker-compose-*-server.yaml up -d --build
   ```

By default, Elasticsearch is deployed as the vector database (`vectordb.yaml` with the default profile). Milvus is optional. You can start Milvus with `docker compose -f deploy/compose/vectordb.yaml --profile milvus up -d` and set `APP_VECTORSTORE_NAME` / `APP_VECTORSTORE_URL` for Milvus (see [Vector database configuration](change-vectordb.md)). When you use the Milvus profile, Milvus can run as GPU-accelerated. Choose the GPU ID using the following variables. For all service port mappings and GPU assignments, refer to [Service Port and GPU Reference](service-port-gpu-reference.md).

   ```bash
   VECTORSTORE_GPU_DEVICE_ID=0
   ```

- For improved accuracy, consider enabling reasoning mode. For details, refer to [Enable thinking](./enable-nemotron-thinking.md).


- Nemotron OCR is now the default OCR service. To use legacy Paddle OCR instead, refer to [OCR Configuration Guide](nemoretriever-ocr.md).

- For advanced users who need direct filesystem access to extraction results, refer to [Ingestor Server Volume Mounting](mount-ingestor-volume.md).

- A single NVIDIA A100-80GB or H100-80GB, B200 GPU can be used to start non-LLM NIMs (nemotron-vlm-embedding-ms, nemotron-ranking-ms, and ingestion services like page-elements, ocr, graphic-elements, and table-structure) for ingestion and RAG workflows. You can control which GPU is used for each service by setting these environment variables in `deploy/compose/.env` file before launching. For a complete list of all services and their default GPU assignments, see [Service Port and GPU Reference](service-port-gpu-reference.md).

   ```bash
   EMBEDDING_MS_GPU_ID=0
   VLM_EMBEDDING_MS_GPU_ID=0
   RANKING_MS_GPU_ID=0
   YOLOX_MS_GPU_ID=0
   YOLOX_GRAPHICS_MS_GPU_ID=0
   YOLOX_TABLE_MS_GPU_ID=0
   OCR_MS_GPU_ID=0
   ```

- If the NIMs are deployed in a different workstation or outside the nvidia-rag docker network on the same system, replace the host address of the below URLs with workstation IPs.

   ```bash
   APP_EMBEDDINGS_SERVERURL="workstation_ip:8000"
   APP_LLM_SERVERURL="workstation_ip:8000"
   APP_RANKING_SERVERURL="workstation_ip:8000"
   OCR_GRPC_ENDPOINT="workstation_ip:8001"
   YOLOX_GRPC_ENDPOINT="workstation_ip:8001"
   YOLOX_GRAPHIC_ELEMENTS_GRPC_ENDPOINT="workstation_ip:8001"
   YOLOX_TABLE_STRUCTURE_GRPC_ENDPOINT="workstation_ip:8001"
   ```



## Related Topics

- [NVIDIA RAG Blueprint Documentation](readme.md)
- [Best Practices for Common Settings](accuracy_perf.md)
- [Multi-Turn Conversation Support](multiturn.md)
- [RAG Pipeline Debugging Guide](debugging.md)
- [Troubleshoot](troubleshooting.md)
- [Notebooks](notebooks.md)
