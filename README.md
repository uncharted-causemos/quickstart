## quickstart
Provide a quick way to run Causemos technology stack. This will build and run Causemos and related services.

The application side covers these services
- https://github.com/uncharted-causemos/causemos
- https://github.com/uncharted-causemos/wm-go
- https://github.com/uncharted-causemos/atlas
- https://github.com/uncharted-causemos/wm-curation-recommendation
- https://github.com/uncharted-causemos/slow-tortoise



### Prerequisite
We need to configure prefect backend and tenant

1. Clone repo and install prefect backend/tenant

```
git clone git@github.com:uncharted-causemos/slow-tortoise.git

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
A docker-compose file is provided here with all the necessary infrastructure for Causemos backend. Note this is quite a heavy stacked and may not perform well on a single computer/laptop.

```
cd infra
docker-compose up
```

### Infrastructure setup/defaults
When the stack is brought up ther are a couple of configurations we need to do:

##### Prefect setup
Goto prefect `http://localhost:8080` and create a new "Production" project

##### Minio setup
Goto minio `http://localhost:9000` and create the following buckets
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


#### Prefect agent (Models and Indicators)
Agents are used to coordinate data ingestion tasks, there are two agents in Causemos: a dask/docker agent and a sequential agent

Dask/docker agent: In a terminal or tmux session, run the prefect docker agent
```
prefect agent docker start \
  --no-pull \
	--api http://localhost:4200 \
	--label wm-prefect-server.openstack.uncharted.software \
	--network common-net \
	--show-flow-logs \
	--env WM_DASK_SCHEDULER=localhost:8786
```


#### Prefect agent (Bring your own data)
Sequential agent: In a terminal or tmux session, run the prefect docker agent

```
TODO
```


### Running the application stack
Make sure the configuration files in `envs` folder are correct and pointing to the right dependencies.

```
cd services
docker-compose up
```
Causemos will be available on `http://localhost:3003`





```
### Loading knowledge data
