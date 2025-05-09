
# Deploying MLExchange Tomo-framework (mlex_tomo_framework)

### Table of Contents  
- [Overview](#Overview)
- [Workflow](#Workflow)
- [Prerequisites](#Prerequisites)
- [Setup-and-Deployment](#Setup-and-Deployment)  
- [Common-Issues](#Common-Issues)

## Overview
The purpose of this document is to assist in the initial deployment and use of the mlex_tomo_framework. This document provides instructions for deployment on Linux devices running podman as the container engine. While the software can run on other operating systems and container engine, it may require different settings from those spelled out here. [Mlex_tomo_framework](https://github.com/mlexchange/mlex_tomo_framework/tree/main) is a framework designed to simplify the deployment and use of the segmentation components of the broader MLExchange platform. It **excludes** the content registry, compute api and other services not necessary for image segmentation.The framework consists of 6 services working and communicating together:  

1. [**Segmentation web-app**](https://github.com/mlexchange/mlex_highres_segmentation): User-facing application for interacting with segmentation results. Allows users to view, initiate and manage segmentation jobs via a web based GUI.  
2. **Prefect Server**: Orchestrates train and inference workloads created by the user in the segmentation app.
3. **PostgreSQL** database to support **prefect**: Stores metadata and state information for the server including details about flow runs and prefect server configuration data  
4. **Tiled Server**: Serves and indexes image data for segmentation workflows.  
5. **PostgreSQL** database to support **tiled**: Backs the Tiled server with persistent storage of dataset, access permissions and configuration settings  
6. **Tiled Ingester**: Consumes ActiveMQ messages and notifies Tiled about new sets of image data  
7. **Prefect Worker**: Tasked by the prefect server to execute the train and inference jobs

## Workflow

### Prerequisites  
- Container Engine: Podman  
- Docker-compose  
- User permissions: Rootless

## Setup-and-Deployment

### Web-App and Server start up  
1. Clone the [mlex_tomo_framework](https://github.com/mlexchange/mlex_tomo_framework) repo.  
2. Ensure podman container engine and docker-compose are working on the system. Enter the following commands into the terminal:
 ```bash
podman -v
docker-compose -v
```
 Both commands should return the version installed. If the terminal returns an error e.g ```podman: command not found``` or ```docker-compose: command not found```, then it means you need to install podman or docker-compose. 
3. Mount the directory that contains the files you want to segment onto the tiled container's ```/tiled_storage/``` direcotry by adding it's path to the volumes section in the docker-compose.yml:  
```yaml
tiled:
	...
	volumes:
		- ./tiled/deploy:/deploy
		- /dls/staging/dls/k11/data/2024/mg37376-1/processing/20240311120527_37086:/tiled_storage:r
```  
4. In the docker-compose.yml, comment out the services for tiled_ingest as such: 
```yaml
	tiled_ingest
```
5. In the docker-compose.yml, in the mlex_segmentation service, change the environment variable DATA_TILED_URI to be ```http://tiled:8000/api/v1/metadata/```.  
6. Replace the prefect_db service's volume with the folowing:
```yaml
volumes:  
	- prefect_db:/var/lib/postgresql/data:rw  
	- ./data/tiled_storage:/tiled_storage
```
7. Replace the tiled_db service's volume with the following:
```yaml
volumes:  
	- tiled_db:/var/lib/postgresql/data:rw
```
8. At the bottom of the docker-compose.yml file, add the following:
```yaml
volumes:  
	prefect_db:
	tiled_db:
```
9. In the .env file, add following enivronment variables:  
```yaml
FLOW_TYPE="podman"  
TRAIN_SCRIPT_PATH="/app/work/src/train.py"  
SEGMENT_SCRIPT_PATH="/app/work/src/segment.py"  
IMAGE_NAME="mlex_dlsia_segmentation"  
IMAGE_TAG="latest"  
CONTAINER_NETWORK="mlex_tomo_framework_mle_net"  
```
10. Start up all the containers with ``` docker-compose up```  
11. Get the id of the tiled container using ```podman ps```
12. Attach to the tiled container using the following command: ```podman exec -it {container_id}  ```
13. In the root directory of the tiled container, run the following command to register all to tiled:  
```tiled catalog register -vw "postgresql+asyncpg://${TILED_DB_USER}:${TILED_DB_PW}@${TILED_DB_SERVER}:5432/${TILED_DB_NAME}" /tiled_storage/ ```
14. If you add new data to the volume you've mounted on tiled through any means other than the mlexchange segmentation UI, you must run the above ```tiled catalog register...``` command again.

### Prefect Worker Startup  
1. In a separate directory, clone the [prefect_worker](https://github.com/mlexchange/mlex_prefect_worker) repo
2. In prefect worker directory, find the .env file and change the ```$CONDA_PATH``` environment variable. Change this to the location of your installation of conda or mamba or any equivalent  
3. Paste these lines to start_worker.sh:
```bash
# The maximum number of flow runs to run simultaneously in the work pool
export  PREFECT_WORK_POOL_CONCURRENCY=4
export  PREFECT_WORKER_LIMIT=4
export  PREFECT_API_URL=http://localhost:4200/api
export  PREFECT_WORK_DIR=$PWD
# Path to conda
export  CONDA_PATH=/dls_sw/apps/mamba/2.0.5
export  PREFECT_WORK_DIR=$PREFECT_WORK_DIR
```
4. In the file flows/podman/podman_flows.py, in the cmd section, replace the ```cmd``` section as follows: 
```python
# Define podman command
cmd = [
"flows/podman/bash_run_podman.sh",
# f"{podman_params.image_name}:{podman_params.image_tag}",
f"mlexchange/{podman_params.image_name}_prototype:main",
# "localhost/test:latest",
command,
" ".join(volumes),
podman_params.network,
" ".join(f"{k}={v}"  for k, v in podman_params.env_vars.items()),
]
logger.info(f"Launching with command: {cmd}")
process =  await run_process(cmd, stream_output=True)
```
5. Ensure you can pull from the github container registry (ghcr.io)
6. Create a conda environment called dlsia with a python version of at least 3.11
7. Activate your conda environment (or any other alternative environment)
8. Run ```start_worker.sh```


## Common-Issues

You can rename the current file by clicking the file name in the navigation bar or by clicking the **Rename** button in the file explorer.
