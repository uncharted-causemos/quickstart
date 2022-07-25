## quickstart
This repository provides a dockerized version of Causemos and its related services. This enables one to rapidly setup and run experiments. There are two main setup script: an application centric that deal with Causemos's sevices, and an infrastructure setup dealing with data storage, schedulers and compute layers. 

Running Causemos stack can be resource intensive, best peroformance is obtained by distributing across a docker-swarm cluster or split the application/infrastucture setups across different machines. On a single machine one will need at least 10GB memory dedicated to Docker.


![system](images/system.png)

The application side covers these services and more detail about individual services can be found here
- https://github.com/uncharted-causemos/causemos
- https://github.com/uncharted-causemos/wm-go
- https://github.com/uncharted-causemos/wm-curation-recommendation
- https://github.com/uncharted-causemos/slow-tortoise
- https://github.com/uncharted-causemos/anansi
- https://github.com/uncharted-causemos/atlas
- https://github.com/uncharted-causemos/wm-request-queue

The infrastructure side covers
- [ElasticSearch](https://www.elastic.co/what-is/elasticsearch)
- [Prefect](https://www.prefect.io/)
- [DASK](https://dask.org/)
- [minio](https://min.io/)


### Requirements
- Docker version 20+
- Python version 3+

### Prerequisite
We need to configure Prefect backend and tenant, this should be on the machine where you want to run the prefect infrastructure.

1. Clone repo and install Prefect backend/tenant

```
git clone git@github.com:uncharted-causemos/slow-tortoise.git

cd slow-tortoise

pip install -e .

prefect backend server

prefect server create-tenant --name default --slug default
```


2. Copy the following to ~/.prefect/config.toml

```
[server]
  [server.ui]
    apollo_url="http://localhost:4200/graphql"

[server.telemetry]
  enabled = false

[engine]
    [engine.executor]
    default_class = "prefect.executors.DaskExecutor"
```


### Running infrastructure stack
A docker-compose file is provided here with all the necessary infrastructure and datastores for Causemos. 

After cloning this repository

```
cd infra

docker-compose up
```

### Infrastructure setup/defaults
After the infrastructure is brought up there are a couple of configurations we need to do:

##### Prefect setup
Go to Prefect `http://localhost:8080` and create a new "Production" project

##### Minio setup
Go to minio `http://localhost:9000` and create the following buckets
 - tiles-v3
 - vector-tiles
 - new-models
 - new-indicators


#### Prefect agent (Models and Indicators)
Agents are used to coordinate data ingestion tasks, there are two agents in Causemos: a dask/docker agent and a sequential agent

Dask/docker agent: In a terminal or tmux session, run the Prefect docker agent
```
prefect agent docker start \
  --no-pull \
  --api http://localhost:4200 \
  --label wm-prefect-server.openstack.uncharted.software \
  --network common-net \
  --show-flow-logs \
  --env WM_DASK_SCHEDULER=localhost:8786
```

#### Register data-pipeline task into Prefect/Dask
Copy the following into `register_datapipeline.sh`.
```
#!/bin/bash

# registration process reads this env var to find the Prefect server
export PREFECT__SERVER__HOST=http://localhost
export DASK_SCHEDULER=localhost:8786
export WM_PUSH_IMAGE=false
export WM_DATA_PIPELINE_IMAGE=uncharted/wm-causemos-data-pipeline

PROJECT="Production"

prefect register --project="$PROJECT" --label wm-prefect-server.openstack.uncharted.software --label docker --path ../flows/data_pipeline.py
```

Then run the script.
```
./register_datapipeline.sh
```

### Setting up concept alignment (optional)

The concept aligner is used to map a string or ontological concept to a datacube in DOJO.

```
# Download docker image
docker pull clulab/conceptalignment:1.2.0

# Run
docker run \
  -p 9001:9001 \
  -e dojo=DOJO_URL \
  -e REST_CONSUMER_ONTOLOGYSERVICE=DART_URL \
  -e REST_CONSUMER_USERNAME=DART_USER \
  -e REST_CONSUMER_PASSWORD=DART_PASSWORD \
  -e secret=SECRET_FOR_WEB_SERVER \
  -e secrets=PASSWORD1|PASSWORD2 \
  -v`pwd`/credentials:/conceptalignment/credentials clulab/conceptalignment:1.2.0
```

The credentials mount should have a `DojoScraper.properties`, with content

```
username = <DOJO_USER>
password = <DOJO_PASSWORD>
```

Note that `secret` is used internally by the web server to prevent abuse and doesn't need to be coordinated with anything else. Just about any string will work.

The other `secrets` need to be coordinated with API clients so that they can have access to the more resource intensive operations like reindexing. You can use multiple strings (separated by a `|`) to differentiate between API users, and pass any one of them along with API calls.

The docker image comes bundled with a default ontology and index of DOJO.

After running the image, you can call the `v2/reindex` endpoint with one of the `secrets` to rescrape and reindex from DOJO, assuming DOJO is up and running.

Similarly, you can call the `/v2/addOntology` endpoint with an ontology ID to load the aligner with a new ontology. This requires DART to be up and running.

Further information can be found in the [concept alignment repo](https://github.com/clulab/ConceptAlignment) and in the Swagger API documentation (http://localhost:9001/api) while running the image.

### Setting up curation/recommendation service (optional)
This is an optional part of Causemos that helps with bulk-curations and CAG building

```
# Get SpaCy model
Download from https://spacy.io/models/en "en_core_web_lg" and extract the tar.gz into the data directory. Note you need `en_core_web_lg-3.0.0`.
```


### Running the application stack
Make sure the configuration files in `envs` folder are correct and pointing to the right dependencies.

```
cd app 
docker-compose up
```
Causemos will be available on `http://localhost:3003`



### Loading knowledge data
Use the curl command to load a knowledge base. Replace the `dart` and `indra` fields as appropriate. 

```
curl -XPOST -H "Content-type: application/json" http://localhost:6000/kb -d'
{
  "indra": "http://10.64.16.209:4005/pipeline-test/indra",
  "dart": "http://10.64.16.209:4005/pipeline-test/dart/july-sample.jsonl"
}
'
```

#### Create embeddings for recommendation service (Optional)
Once a INDRA dataset has been ingested, there will be an index `indra-{uuid}` identifier denoting the index created in ElasticSearch. We can then use this to create the recommender indices. Assuming wm-curation-recommendation service is running.

Note if we are using the ElasticSearch inside a docker-compose, the `es_url` needs to be reachable, e.g.  `"es_url": "http://elasticsearch:9200"`.

```
curl -H "Content-type:application/json" -XPOST http://<curation_server>:<port>/recommendation/ingest/{indra_index} -d'
{
  "remove_factors": true,
  "remove_statements": true,
  "es_url": {destination_es}:9200
}
'
```

This will yield a task id while the indices are build asynchronously in the background. We can check the build status with 

```
curl http://{recommendation_server}:{port}/recommendation/task/{task_id}
```
