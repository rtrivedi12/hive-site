---
title: "Apache Hive : Setting Up Hive with Docker"
date: 2024-12-12
---

# Apache Hive : Setting Up Hive with Docker

### Introduction

---

Run Apache Hive inside docker container in pseudo-distributed mode

##### **STEP 1: Pull the image**

* Pull the **4.0.0** image from [Hive DockerHub](https://hub.docker.com/r/apache/hive/tags)

```
docker pull apache/hive:4.0.0

```
##### **STEP 2: Export the Hive version**

```
export HIVE_VERSION=4.0.0

```
##### **STEP 3: Launch the HiveServer2 with an embedded Metastore.**

This is lightweight and for a quick setup, it uses Derby as metastore db.

```
docker run -d -p 10000:10000 -p 10002:10002 --env SERVICE_NAME=hiveserver2 --name hive4 apache/hive:${HIVE_VERSION}

```
##### **STEP 4: Connect to beeline**

```
docker exec -it hiveserver2 beeline -u 'jdbc:hive2://hiveserver2:10000/'

```
Note: Launch Standalone Metastore To use standalone Metastore with Derby,

```
docker run -d -p 9083:9083 --env SERVICE_NAME=metastore --name metastore-standalone apache/hive:${HIVE_VERSION}

```
## Advanced Setup

### Run services

##### **- Metastore**

For a quick start, launch the Metastore with Derby,

```
docker run -d -p 9083:9083 --env SERVICE_NAME=metastore --name metastore-standalone apache/hive:4.0.0

```
Everything would be lost when the service is down. In order to save the Hive table's schema and data, start the container with an external Postgres and Volume to keep them,

```
docker run -d -p 9083:9083 --env SERVICE_NAME=metastore --env DB_DRIVER=postgres \  
     --env SERVICE_OPTS="-Djavax.jdo.option.ConnectionDriverName=org.postgresql.Driver -Djavax.jdo.option.ConnectionURL=jdbc:postgresql://postgres:5432/metastore\_db -Djavax.jdo.option.ConnectionUserName=hive -Djavax.jdo.option.ConnectionPassword=password" \  
     --mount source=warehouse,target=/opt/hive/data/warehouse \  
     --mount type=bind,source=`mvn help:evaluate -Dexpression=settings.localRepository -q -DforceStdout`/org/postgresql/postgresql/42.5.1/postgresql-42.5.1.jar,target=/opt/hive/lib/postgres.jar \  
     --name metastore-standalone apache/hive:4.0.0
```
If you want to use your own `hdfs-site.xml` or `yarn-site.xml` for the service, you can provide the environment variable `HIVE_CUSTOM_CONF_DIR` for the command. For instance, put the custom configuration file under the directory `/opt/hive/conf`, then run,

```
docker run -d -p 9083:9083 --env SERVICE\_NAME=metastore --env DB\_DRIVER=postgres \
     -v /opt/hive/conf:/hive\_custom\_conf --env HIVE\_CUSTOM\_CONF\_DIR=/hive\_custom\_conf \
     --mount type=bind,source=`mvn help:evaluate -Dexpression=settings.localRepository -q -DforceStdout`/org/postgresql/postgresql/42.5.1/postgresql-42.5.1.jar,target=/opt/hive/lib/postgres.jar \
     --name metastore apache/hive:4.0.0  
  
For Hive releases before 4.0, if you want to upgrade the existing external Metastore schema to the target version, then add --env SCHEMA\_COMMAND=upgradeSchema to the command.  
To skip schematool initialisation or upgrade for metastore use --env `IS_RESUME="true"`, and for verbose logging set --env `VERBOSE="true".`
```
##### **- HiveServer2**

Launch the HiveServer2 with an embedded Metastore,

```
 docker run -d -p 10000:10000 -p 10002:10002 --env SERVICE_NAME=hiveserver2 --name hiveserver2-standalone apache/hive:4.0.0

```
or specify a remote Metastore if it's available,

```
 docker run -d -p 10000:10000 -p 10002:10002 --env SERVICE_NAME=hiveserver2 \
      --env SERVICE_OPTS="-Dhive.metastore.uris=thrift://metastore:9083" \
      --env IS_RESUME="true" \
      --name hiveserver2-standalone apache/hive:4.0.0

```
To save the data between container restarts, you can start the HiveServer2 with a Volume,

```
 docker run -d -p 10000:10000 -p 10002:10002 --env SERVICE_NAME=hiveserver2 \
      --env SERVICE_OPTS="-Dhive.metastore.uris=thrift://metastore:9083" \
      --mount source=warehouse,target=/opt/hive/data/warehouse \
      --env IS_RESUME="true" \
      --name hiveserver2 apache/hive:4.0.0  

```
##### **- HiveServer2, Metastore**

To get a quick overview of both HiveServer2 and Metastore, there is a `docker-compose.yml` placed under `packaging/src/docker` for this purpose, specify the `POSTGRES_LOCAL_PATH` first:

```
export POSTGRES\_LOCAL\_PATH=your\_local\_path\_to\_postgres\_driver
```
Example:

```
mvn dependency:copy -Dartifact="org.postgresql:postgresql:42.5.1" && \
export POSTGRES\_LOCAL\_PATH=`mvn help:evaluate -Dexpression=settings.localRepository -q -DforceStdout`/org/postgresql/postgresql/42.5.1/postgresql-42.5.1.jar 
```
If you don't install maven or have problem in resolving the postgres driver, you can always download this jar yourself, change the `POSTGRES_LOCAL_PATH` to the path of the downloaded jar.

Then,

```
docker compose up -d
```
HiveServer2, Metastore and Postgres services will be started as a consequence. Volumes are used to persist data generated by Hive inside Postgres and HiveServer2 containers,

* hive\_db

The volume persists the metadata of Hive tables inside Postgres container.
* warehouse

The volume stores tables' files inside HiveServer2 container.

To stop/remove them all,

```
docker compose down
```
### Usage

* HiveServer2 web
	+ Accessed on browser at http://localhost:10002/
* Beeline:
```
  docker exec -it hiveserver2 beeline -u 'jdbc:hive2://hiveserver2:10000/'
  # If beeline is installed on host machine, HiveServer2 can be simply reached via:
  beeline -u 'jdbc:hive2://localhost:10000/'

```
* Run some queries
```
  show tables;
  create table hive_example(a string, b int) partitioned by(c int);
  alter table hive_example add partition(c=1);
  insert into hive_example partition(c=1) values('a', 1), ('a', 2),('b',3);
  select count(distinct a) from hive_example;
  select sum(b) from hive_example;

```

 

 
