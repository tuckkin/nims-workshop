# Lab 2 - Deployment Enterprise RAG and Agentic RAG

In this Hands-On Lab you deploy the NVIDIA RAG Blueprint with Docker Compose for a single node deployment, and using NVIDIA-hosted LLM model on cloud.

## 2.1 Deploy Enterprise RAG
### 2.1.1 Register for NVIDIA account and request for API Key

1. Go to https://build.nvidia.com/ and login using your NVIDIA account.

2. Create an API Key
    Refer to https://docs.nvidia.com/nim/large-language-models/latest/getting-started.html#launch-nvidia-nim-for-llms

3. Using the API Key, perform docker login to nvcr.io
    ```bash
    export NGC_API_KEY="<YOUR_NGC_API_KEY>"
    echo "$NGC_API_KEY" | docker login nvcr.io --username '$oauthtoken' --password-stdin
    ```

### 2.1.2 Access to the local RAG Blueprint Repository

The RAG Blueprint was downloaded from `https://github.com/NVIDIA-AI-Blueprints/rag.git` to the local folder:

1. Access to the cloned copy of the RAG Blueprint:  

   ```bash
   cd ~/workspace/rag
   ``` 

### 2.1.3 Start Enterprise RAG services

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

5. Check the status of the deployment by running the following code. Wait until all services are up and the `nemotron-ranking-ms` and `nemotron-vlm-embedding-ms`, and `nim-llm-ms` NIMs are in healthy state before proceeding further.<br><br>
<b>Note</b>: Most of the RAG components, including the Ranking and Embedding models will be running on-premise, except for LLM model `nim-llm-ms` which is the NVIDIA-hosted LLM model on cloud due to GPU resource constraint

     ```bash
     watch -n 2 'docker ps --format "table {{.Names}}\t{{.Status}}"'
     ```
    Your output should look similar to the following.

     ```output
        NAMES                         STATUS

        nemotron-ranking-ms           Up 4 minutes (healthy)
        compose-graphic-elements-1    Up 4 minutes
        compose-page-elements-1       Up 4 minutes
        nemotron-vlm-embedding-ms     Up 4 minutes (healthy)
        compose-nemotron-ocr-1        Up 4 minutes
        compose-table-structure-1     Up 4 minutes
     ```

6. Start the vector db containers from the repo root.

   ```bash
   docker compose -f deploy/compose/vectordb.yaml up -d
   ```

7. Start the ingestion containers from the repo root. This pulls the prebuilt containers from NGC and deploys them on your system.

   ```bash
   docker compose -f deploy/compose/docker-compose-ingestor-server.yaml up -d
   ```

8. Start the RAG containers from the repo root. This pulls the prebuilt containers from NGC and deploys them on your system.

    ```bash
    docker compose -f deploy/compose/docker-compose-rag-server.yaml up -d
    ```

9. Check the status of the deployment by running the following code.

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
    fe2751bfa734   nemotron-ranking-ms              Up 10 minutes (healthy)
    7b5ddabf8be7   compose-graphic-elements-1       Up 10 minutes
    ecfaa5190302   compose-page-elements-1          Up 10 minutes
    ea8c7fdf20d1   nemotron-vlm-embedding-ms        Up 10 minutes (healthy)
    6d62008a9b42   compose-nemotron-ocr-1           Up 10 minutes
    969b9f5c987c   compose-table-structure-1        Up 10 minutes
    ```

### 2.1.4 Experiment with the Web User Interface

After the RAG Blueprint is deployed, you can use the RAG UI to start experimenting with it.

1. Open a web browser and access the RAG UI. You can start experimenting by uploading docs and asking questions. For details, see [User Interface for NVIDIA RAG Blueprint](user-interface.md).

## 2.3 Shut down services

1. To stop all running services, run the following code.

    ```bash
    docker compose -f deploy/compose/docker-compose-ingestor-server.yaml down
    docker compose -f deploy/compose/nims.yaml down
    docker compose -f deploy/compose/docker-compose-rag-server.yaml down
    docker compose -f deploy/compose/vectordb.yaml down
    docker volume ls -q --filter "name=^rag-vol-" | xargs -r docker volume rm
    ```
## 2.4 Related Topics

- [NVIDIA RAG Blueprint Documentation](readme.md)
- [Best Practices for Common Settings](accuracy_perf.md)
- [Multi-Turn Conversation Support](multiturn.md)
- [RAG Pipeline Debugging Guide](debugging.md)
- [Troubleshoot](troubleshooting.md)
- [Notebooks](notebooks.md)
