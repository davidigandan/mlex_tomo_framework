<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Installation instructions (non-diamond).md</title>
  <link rel="stylesheet" href="https://stackedit.io/style.css" />
</head>

<body class="stackedit">
  <div class="stackedit__left">
    <div class="stackedit__toc">
      
<ul>
<li><a href="#deploying-mlexchange-tomo-framework-mlex_tomo_framework">Deploying MLExchange Tomo-framework (mlex_tomo_framework)</a>
<ul>
<li></li>
<li><a href="#overview">Overview</a></li>
<li><a href="#environment-setup-and-deployment">Environment Setup and Deployment</a></li>
<li><a href="#common-issues--troubleshooting">Common Issues & Troubleshooting</a></li>
</ul>
</li>
</ul>

    </div>
  </div>
  <div class="stackedit__right">
    <div class="stackedit__html">
      <h1 id="deploying-mlexchange-tomo-framework-mlex_tomo_framework">Deploying MLExchange Tomo-framework (mlex_tomo_framework)</h1>
<h3 id="table-of-contents">Table of Contents</h3>
<ul>
<li><a href="##Overview">Overview</a></li>
<li>[The Workflow](##The Workflow)</li>
<li><a href="##Prerequisites">Prerequisites</a></li>
<li>[Environment Setup &amp; Deployment](##Environment Setup and Deployment)</li>
<li>[Common Issues &amp; Troubleshooting](##Common Issues &amp; Troubleshooting)</li>
</ul>
<h2 id="overview">Overview</h2>
<p>The purpose of this document is to assist in the initial deployment and use of the mlex_tomo_framework. This document provides instructions for deployment on Linux devices running podman as the container engine. While the software can run on other operating systems and container engine, it may require different settings from those spelled out here. <a href="https://github.com/mlexchange/mlex_tomo_framework/tree/main">Mlex_tomo_framework</a> is a framework designed to simplify the deployment and use of the segmentation components of the broader MLExchange platform. It <strong>excludes</strong> the content registry, compute api and other services not necessary for image segmentation.The framework consists of 6 services working and communicating together:</p>
<ol>
<li><strong>Segmentation web-app</strong>: User-facing application for interacting with segmentation results. Allows users to view, initiate and manage segmentation jobs via a web based GUI.</li>
<li><strong>Prefect Server</strong>: Orchestrates train and inference workloads created by the user in the segmentation app.</li>
<li><strong>PostgreSQL</strong> database to support <strong>prefect</strong>: Stores metadata and state information for the server including details about flow runs and prefect server configuration data</li>
<li><strong>Tiled Server</strong>: Serves and indexes image data for segmentation workflows.</li>
<li><strong>PostgreSQL</strong> database to support <strong>tiled</strong>: Backs the Tiled server with persistent storage of dataset, access permissions and configuration settings</li>
<li><strong>Tiled Ingester</strong>: Consumes ActiveMQ messages and notifies Tiled about new sets of image data</li>
<li><strong>Prefect Worker</strong>: Tasked by the prefect server to execute the train and inference jobs</li>
</ol>
<h3 id="prerequisites">Prerequisites</h3>
<ul>
<li>Container Engine: Podman</li>
<li>Docker-compose</li>
<li>User permissions: Rootless</li>
</ul>
<h2 id="environment-setup-and-deployment">Environment Setup and Deployment</h2>
<h3 id="web-app-and-server-start-up">Web-App and Server start up</h3>
<ol>
<li>Clone the <a href="https://github.com/mlexchange/mlex_tomo_framework">mlex_tomo_framework</a> repo.</li>
<li>Ensure podman container engine and docker-compose are working on the system. Enter the following commands into the terminal:</li>
</ol>
<pre class=" language-bash"><code class="prism  language-bash">podman -v
docker-compose -v
</code></pre>
<p>Both commands should return the version installed. If the terminal returns an error e.g <code>podman: command not found</code> or <code>docker-compose: command not found</code>, then it means you need to install podman or docker-compose.<br>
3. Mount the directory that contains the files you want to segment onto the tiled container’s <code>/tiled_storage/</code> direcotry by adding it’s path to the volumes section in the docker-compose.yml:</p>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">tiled</span><span class="token punctuation">:</span>
	<span class="token punctuation">...</span>
	<span class="token key atrule">volumes</span><span class="token punctuation">:</span>
		<span class="token punctuation">-</span> ./tiled/deploy<span class="token punctuation">:</span>/deploy
		<span class="token punctuation">-</span> /dls/staging/dls/k11/data/2024/mg37376<span class="token punctuation">-</span>1/processing/20240311120527_37086<span class="token punctuation">:</span>/tiled_storage<span class="token punctuation">:</span>r
</code></pre>
<ol start="4">
<li>In the docker-compose.yml, comment out the services for tiled_ingest as such:</li>
</ol>
<pre class=" language-yaml"><code class="prism  language-yaml">	tiled_ingest
</code></pre>
<ol start="5">
<li>In the docker-compose.yml, in the mlex_segmentation service, change the environment variable DATA_TILED_URI to be <code>http://tiled:8000/api/v1/metadata/</code>.</li>
<li>Replace the prefect_db service’s volume with the folowing:</li>
</ol>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">volumes</span><span class="token punctuation">:</span>  
	<span class="token punctuation">-</span> prefect_db<span class="token punctuation">:</span>/var/lib/postgresql/data<span class="token punctuation">:</span>rw  
	<span class="token punctuation">-</span> ./data/tiled_storage<span class="token punctuation">:</span>/tiled_storage
</code></pre>
<ol start="7">
<li>Replace the tiled_db service’s volume with the following:</li>
</ol>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">volumes</span><span class="token punctuation">:</span>  
	<span class="token punctuation">-</span> tiled_db<span class="token punctuation">:</span>/var/lib/postgresql/data<span class="token punctuation">:</span>rw
</code></pre>
<ol start="8">
<li>At the bottom of the docker-compose.yml file, add the following:</li>
</ol>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">volumes</span><span class="token punctuation">:</span>  
	<span class="token key atrule">prefect_db</span><span class="token punctuation">:</span>
	<span class="token key atrule">tiled_db</span><span class="token punctuation">:</span>
</code></pre>
<ol start="9">
<li>In the .env file, add following enivronment variables:</li>
</ol>
<pre class=" language-yaml"><code class="prism  language-yaml">FLOW_TYPE="podman"  
TRAIN_SCRIPT_PATH="/app/work/src/train.py"  
SEGMENT_SCRIPT_PATH="/app/work/src/segment.py"  
IMAGE_NAME="mlex_dlsia_segmentation"  
IMAGE_TAG="latest"  
CONTAINER_NETWORK="mlex_tomo_framework_mle_net"  
</code></pre>
<ol start="10">
<li>Start up all the containers with <code>docker-compose up</code></li>
<li>Get the id of the tiled container using <code>podman ps</code></li>
<li>Attach to the tiled container using the following command: <code>podman exec -it {container_id}</code></li>
<li>In the root directory of the tiled container, run the following command to register all to tiled:<br>
<code>tiled catalog register -vw "postgresql+asyncpg://${TILED_DB_USER}:${TILED_DB_PW}@${TILED_DB_SERVER}:5432/${TILED_DB_NAME}" /tiled_storage/</code></li>
<li>If you add new data to the volume you’ve mounted on tiled through any means other than the mlexchange segmentation UI, you must run the above <code>tiled catalog register...</code> command again.</li>
</ol>
<h3 id="prefect-worker-startup">Prefect Worker Startup</h3>
<ol>
<li>In a separate directory, clone the <a href="https://github.com/mlexchange/mlex_prefect_worker">prefect_worker</a> repo</li>
<li>In prefect worker directory, find the .env file and change the <code>$CONDA_PATH</code> environment variable. Change this to the location of your installation of conda or mamba or any equivalent</li>
<li>Paste these lines to start_worker.sh:</li>
</ol>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token comment"># The maximum number of flow runs to run simultaneously in the work pool</span>
<span class="token function">export</span>  PREFECT_WORK_POOL_CONCURRENCY<span class="token operator">=</span>4
<span class="token function">export</span>  PREFECT_WORKER_LIMIT<span class="token operator">=</span>4
<span class="token function">export</span>  PREFECT_API_URL<span class="token operator">=</span>http://localhost:4200/api
<span class="token function">export</span>  PREFECT_WORK_DIR<span class="token operator">=</span><span class="token variable">$PWD</span>
<span class="token comment"># Path to conda</span>
<span class="token function">export</span>  CONDA_PATH<span class="token operator">=</span>/dls_sw/apps/mamba/2.0.5
<span class="token function">export</span>  PREFECT_WORK_DIR<span class="token operator">=</span><span class="token variable">$PREFECT_WORK_DIR</span>
</code></pre>
<ol start="4">
<li>In the file flows/podman/podman_flows.py, in the cmd section, replace the <code>cmd</code> section as follows:</li>
</ol>
<pre class=" language-python"><code class="prism  language-python"><span class="token comment"># Define podman command</span>
cmd <span class="token operator">=</span> <span class="token punctuation">[</span>
<span class="token string">"flows/podman/bash_run_podman.sh"</span><span class="token punctuation">,</span>
<span class="token comment"># f"{podman_params.image_name}:{podman_params.image_tag}",</span>
f<span class="token string">"mlexchange/{podman_params.image_name}_prototype:main"</span><span class="token punctuation">,</span>
<span class="token comment"># "localhost/test:latest",</span>
command<span class="token punctuation">,</span>
<span class="token string">" "</span><span class="token punctuation">.</span>join<span class="token punctuation">(</span>volumes<span class="token punctuation">)</span><span class="token punctuation">,</span>
podman_params<span class="token punctuation">.</span>network<span class="token punctuation">,</span>
<span class="token string">" "</span><span class="token punctuation">.</span>join<span class="token punctuation">(</span>f<span class="token string">"{k}={v}"</span>  <span class="token keyword">for</span> k<span class="token punctuation">,</span> v <span class="token keyword">in</span> podman_params<span class="token punctuation">.</span>env_vars<span class="token punctuation">.</span>items<span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">,</span>
<span class="token punctuation">]</span>
logger<span class="token punctuation">.</span>info<span class="token punctuation">(</span>f<span class="token string">"Launching with command: {cmd}"</span><span class="token punctuation">)</span>
process <span class="token operator">=</span>  <span class="token keyword">await</span> run_process<span class="token punctuation">(</span>cmd<span class="token punctuation">,</span> stream_output<span class="token operator">=</span><span class="token boolean">True</span><span class="token punctuation">)</span>
</code></pre>
<ol start="5">
<li>Ensure you can pull from the github container registry (<a href="http://ghcr.io">ghcr.io</a>)</li>
<li>Create a conda environment called dlsia with a python version of at least 3.11</li>
<li>Activate your conda environment (or any other alternative environment)</li>
<li>Run <code>start_worker.sh</code></li>
</ol>
<h2 id="common-issues--troubleshooting">Common Issues &amp; Troubleshooting</h2>
<p>You can rename the current file by clicking the file name in the navigation bar or by clicking the <strong>Rename</strong> button in the file explorer.</p>

    </div>
  </div>
</body>

</html>
