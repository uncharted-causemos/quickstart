## quickstart
This repository provides a dockerized version of Causemos and its related services. This enables one to rapidly setup and run experiments. There are two main setup script: an application centric that deal with Causemos's sevices, and an infrastructure setup dealing with data storage, schedulers and compute layers. 

Running Causemos stack can be resource intensive, best peroformance is obtained by distributing across a docker-swarm cluster or run application/infrastucture setups on two different machine. On a single machine one will need at least 8GB dedicated to Docker.



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
We need to configure Prefect backend and tenant

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
A docker-compose file is provided here with all the necessary infrastructure for Causemos backend. Note this is quite a heavy stack and may not perform well on a single computer/laptop.

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


##### ElasticSearch
Create initial mapping schema for ES datastore
```
git clone git@github.com:uncharted-causemos/atlas.git

cd atlas

ES=<elastic_url> python ./es_mapper.py
```

##### Configure geo reference dataset
Download and extract the following geolocation datasets:
- http://download.geonames.org/export/dump/allCountries.zip
- http://clulab.cs.arizona.edu/models/gadm_woredas.txt

Then use anansi utility to load the dataset
```
# 1. Clone or switch over to anansi
git clone git@github.com:uncharted-causemos/anansi.git

# 2. Copy the two data sets to src directory of anansi

# 3. In src, run
ES=<es_url> ES_USER=<user> ES_PASSWORD=<password> python geo_loader.py
```


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

# set this to true if images should be pushed to the docker registry as part of the
# registration process - not necessary if testing locally
export WM_PUSH_IMAGE=true

PROJECT="Production"

# add calls to register flows here
prefect register --project="$PROJECT" --label wm-prefect-server.openstack.uncharted.software --label docker --path ../flows/data_pipeline.py
```

### Setting up concept alignment

The concept aligner is used to map a string or ontological concept to a datacube in Dojo.

```
# Download docker image
docker pull clulab/conceptalignment:1.2.0

# Run
docker run -p 9001:9001 --name conceptalignment -e secret="<secret_for_web_server>" -e secrets="password1|password2" -v`pwd`/../credentials:/conceptalignment/credentials clulab/conceptalignment:1.2.0.
```

Note that `secret` is used internally by the web server to prevent some kinds of abuse and doesn't need to be coordinated with anything else. Just about any string will work.

The other `secrets` need to be coordinated with API clients so that they can have access to the more resource intensive operations like reindexing. You can use multiple strings (separated by a `|`) to differentiate between API users, and pass any one of them along with API calls.

The docker image comes bundled with a default ontology and index of Dojo.

After running the image, you can call the `v2/reindex` endpoint with one of the `secrets` to rescrape and reindex from Dojo, assuming Dojo is up and running.

Similarly, you can call the `/v2/addOntology` endpoint with an ontology ID to load the aligner with a new ontology. This requires DART to be up and running.

Further information can be found in the [concept alignment repo](https://github.com/clulab/ConceptAlignment) and in the Swagger API documentation while running the image.

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
Data loaders are in the [anansi repository](https://github.com/uncharted-causemos/anansi).

```
git clone git@github.com:uncharted-causemos/anansi.git
```

Please see the following two scripts for acquiring INDRA and DART datasets from running services. From here on out we will assume these two datasets are available. 
- scripts/build_dart.sh
- scripts/download_indra_s3.py 



To create a knowledge-base we can run the following, for all intents and purposes here SOURCE_* and TARGET_* are the same.

```
#!/usr/bin/env bash

SOURCE_ES=xyz \
SOURCE_USERNAME=xyz \
SOURCE_PASSWORD=xyz \
TARGET_ES=xyz \
TARGET_USERNAME=xyz \
TARGET_PASSWORD=xyz \
DART_DATA=<path_to_dart_cdr.json> \
INDRA_DATASET=<path_to_indra_directory> \
python src/knowledge_pipeline.py
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



### Bring-your-own-data
The bring-your-own-data feature requires installing a Prefect task and agent. This assumes that both DART and INDRA are running as services.


```
git clone git@github.com:uncharted-causemos/anansi.git
```

#### Prefect agent

Create an `incremental.env` file, and populate the values
```
export SOURCE_ES=
export SOURCE_USERNAME=
export SOURCE_PASSWORD=
export TARGET_ES=
export TARGET_USERNAME=
export TARGET_PASSWORD=
export DART_HOST=
export DART_USER=
export DART_PASS=
export INDRA_HOST=
export CURATION_HOST=http://localhost:5000
```

Register the flow with Prefect
```
source incremental.env

PREFECT__ENGINE__EXECUTOR__DEFAULT_CLASS="prefect.executors.LocalExecutor" PYTHONPATH="${PYTHONPATH}:./src" prefect agent local start --api "http://localhost:4200/graphql" --label "non-dask"
```

In a terminal or tmux session, run the Prefect docker agent.


#### Registering task
Note: You need to edit `src/incremental_pipeline.py` and change the variable `should_register` to True
```
python incremental_pipeline.py
```


