<!--
{% comment %}
Licensed to the Apache Software Foundation (ASF) under one or more
contributor license agreements.  See the NOTICE file distributed with
this work for additional information regarding copyright ownership.
The ASF licenses this file to you under the Apache License, Version 2.0
(the "License"); you may not use this file except in compliance with
the License.  You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
{% endcomment %}
-->

#hive-ub8
### Introduction

---
Rebuild of Apache Hive docker container on UBI8.

Goals:
Keep with modern dependency/ patches.
Run on UBI8


Run Apache Hive inside docker container in pseudo-distributed mode, provide the following
- Quick-start/Debugging/Prepare a test env for Hive


## Quickstart
### STEP 1: Pull the image
- Pull the image from DockerHub: https://hub.docker.com/r/aquabear/hive-ubi8/tags. 

Here are the latest images:
- 4.0.0


```shell
docker pull aquabear/hive-ubi8:4.0.0
```
### STEP 2: Export the Hive version
```shell
export HIVE_VERSION=4.0.0
```

### STEP 3: Launch the HiveServer2 with an embedded Metastore.
This is lightweight and for a quick setup, it uses Derby as metastore db.
```shell
docker run -d -p 10000:10000 -p 10002:10002 --env SERVICE_NAME=hiveserver2 --name hive4 aquabear/hive-ubi8:${HIVE_VERSION}
```

### STEP 4: Connect to beeline
```shell
docker exec -it hiveserver2 beeline -u 'jdbc:hive2://hiveserver2:10000/'
```

#### Note: Launch Standalone Metastore To use standalone Metastore with Derby,
```shell
docker run -d -p 9083:9083 --env SERVICE_NAME=metastore --name metastore-standalone aquabear/hive-ubi8:${HIVE_VERSION}
```

### Detailed Setup
---
#### Build image
Apache Hive relies on Hadoop, Tez and some others to facilitate reading, writing, and managing large datasets. 
The `build.sh` provides ways to build the image against specified version of the dependent, as well as build from source.


```
##### Build with specified version
There are some arguments to specify the component version:
```shell
-hadoop <hadoop version>
-tez <tez version>
-hive <hive version> 
```
If the version is not provided, it will read the version from current `build.sh`:
`hive_version`, `hadoop_version` and `tez_version` for Hive, Hadoop and Tez respectively. 

For example, the following command uses Hive 4.0.0, Hadoop `hadoop.version` and Tez `tez.version` to build the image,
```shell
./build.sh -hive 4.0.0
```
If the command does not specify the Hive version, it will use the local `apache-hive-${project.version}-bin.tar.gz`(will trigger a build if it doesn't exist),
together with Hadoop 3.1.0 and Tez 0.10.1 to build the image,
```shell
./build.sh -hadoop 3.1.0 -tez 0.10.1
```
After building successfully,  we can get a Docker image named `aquabear/hive-ubi8` by default, the image is tagged by the provided Hive version.

#### Run services

Before going further, we should define the environment variable `HIVE_VERSION` first.
For example, if `-hive 4.0.0` is specified to build the image,
```shell
export HIVE_VERSION=4.0.0
```

- Dev Example
  Using External MariaDB VM or Container - Dev Example

T/S Command Version to Skip Entrypoint and go to bash.
  ```shell
    docker run -p 9083:9083 -p 10000:10000 -p 10002:10002 -it --entrypoint /bin/bash -h "hiveserver2" --env SERVICE_NAME=hiveserver2 --env DB_DRIVER=mysql --env IS_RESUME="true" --env SERVICE_OPTS="-Djavax.jdo.option.ConnectionDriverName=org.mariadb.jdbc.Driver -Djavax.jdo.option.ConnectionURL=jdbc:mariadb://10.10.15.121:3306/hive -Djavax.jdo.option.ConnectionUserName=hive -Djavax.jdo.option.ConnectionPassword=hive" --mount type=bind,source=/opt/supportfiles/mariadb-java-client-3.3.3.jar,target=/opt/hive/lib/mariadb.jar -v /mnt/smb/data/hive_data:/opt/hive/data -v /mnt/smb/data/hive_conf/hive:/opt/hive/conf -v /mnt/smb/data/hive_conf/tez:/opt/tez/conf -v /mnt/smb/data/hive_conf/hadoop:/opt/hadoop/etc/hadoop --user 1000:1000 -v /mnt/smb/data/warehouse,target=/opt/hive/data/warehouse --name hiveserver2 aquabear/hive-ubi8:4.0.0
  ```	
  
Launch with MariaDB VM
  ```shell
	docker run -d -p 9083:9083 -p 10000:10000 -p 10002:10002 --env SERVICE_NAME=hiveserver2 -h "hiveserver2" --env DB_DRIVER=mysql --env IS_RESUME="true" --env SERVICE_OPTS="-Djavax.jdo.option.ConnectionDriverName=org.mariadb.jdbc.Driver -Djavax.jdo.option.ConnectionURL=jdbc:mariadb://10.10.15.121:3306/hive -Djavax.jdo.option.ConnectionUserName=hive -Djavax.jdo.option.ConnectionPassword=hive" --mount type=bind,source=/opt/supportfiles/mariadb-java-client-3.3.3.jar,target=/opt/hive/lib/mariadb.jar -v /mnt/smb/data/hive_data:/opt/hive/data -v /mnt/smb/data/hive_conf/hive:/opt/hive/conf -v /mnt/smb/data/hive_conf/tez:/opt/tez/conf -v /mnt/smb/data/hive_conf/hadoop:/opt/hadoop/etc/hadoop --user 1000:1000 -v /mnt/smb/data/warehouse,target=/opt/hive/data/warehouse --name hiveserver2 aquabear/hive-ubi8:4.0.0

  ```

- Metastore

For a quick start, launch the Metastore with Derby,
  ```shell
  docker run -d -p 9083:9083 --env SERVICE_NAME=metastore --name metastore-standalone aquabear/hive-ubi8:${HIVE_VERSION}
  ```
  Everything would be lost when the service is down. In order to save the Hive table's schema and data, start the container with an external Postgres and Volume to keep them,

  ```shell
  docker run -d -p 9083:9083 --env SERVICE_NAME=metastore --env DB_DRIVER=postgres \
       --env SERVICE_OPTS="-Djavax.jdo.option.ConnectionDriverName=org.postgresql.Driver -Djavax.jdo.option.ConnectionURL=jdbc:postgresql://postgres:5432/metastore_db -Djavax.jdo.option.ConnectionUserName=hive -Djavax.jdo.option.ConnectionPassword=password" \
       --mount source=warehouse,target=/opt/hive/data/warehouse \
       --mount type=bind,source=`mvn help:evaluate -Dexpression=settings.localRepository -q -DforceStdout`/org/postgresql/postgresql/42.7.3/postgresql-42.7.3.jar,target=/opt/hive/lib/postgres.jar \
       --name metastore-standalone aquabear/hive-ubi8:${HIVE_VERSION}
  ```

  If you want to use your own `hdfs-site.xml` or `yarn-site.xml` for the service, you can provide the environment variable `HIVE_CUSTOM_CONF_DIR` for the command. For instance, put the custom configuration file under the directory `/opt/hive/conf`, then run,

  ```shell
   docker run -d -p 9083:9083 --env SERVICE_NAME=metastore --env DB_DRIVER=postgres \
        -v /opt/hive/conf:/hive_custom_conf --env HIVE_CUSTOM_CONF_DIR=/hive_custom_conf \
        --mount type=bind,source=`mvn help:evaluate -Dexpression=settings.localRepository -q -DforceStdout`/org/postgresql/postgresql/42.7.3/postgresql-42.7.3.jar,target=/opt/hive/lib/postgres.jar \
        --name metastore aquabear/hive-ubi8:${HIVE_VERSION}
  ```

  Using External MariaDB VM or Container - Dev Example

  ```shell
   docker run -d -p 9083:9083 --env SERVICE_NAME=metastore --env DB_DRIVER=postgres \
        -v /opt/hive/conf:/hive_custom_conf --env HIVE_CUSTOM_CONF_DIR=/hive_custom_conf \
        --mount type=bind,source=`mvn help:evaluate -Dexpression=settings.localRepository -q -DforceStdout`/org/postgresql/postgresql/42.7.3/postgresql-42.7.3.jar,target=/opt/hive/lib/postgres.jar \
        --name metastore aquabear/hive-ubi8:${HIVE_VERSION}
  ```


NOTE:

For Hive releases before 4.0, if you want to upgrade the existing external Metastore schema to the target version,
then add "--env SCHEMA_COMMAND=upgradeSchema" to the command.

- HiveServer2

Launch the HiveServer2 with an embedded Metastore,
   ```shell
  docker run -d -p 10000:10000 -p 10002:10002 --env SERVICE_NAME=hiveserver2 --name hiveserver2-standalone aquabear/hive-ubi8:${HIVE_VERSION}
   ```
  or specify a remote Metastore if it's available,
   ```shell
    docker run -d -p 10000:10000 -p 10002:10002 --env SERVICE_NAME=hiveserver2 \
         --env SERVICE_OPTS="-Dhive.metastore.uris=thrift://metastore:9083" \
         --env IS_RESUME="true" \
         --env VERBOSE="true" \
         --name hiveserver2-standalone aquabear/hive-ubi8:${HIVE_VERSION}
   ```

NOTE:

To skip schematool initialisation or upgrade for metastore use `IS_RESUME="true"`, and for verbose logging set `VERBOSE="true"`


  To save the data between container restarts, you can start the HiveServer2 with a Volume,

   ```shell
   docker run -d -p 10000:10000 -p 10002:10002 --env SERVICE_NAME=hiveserver2 \
      --env SERVICE_OPTS="-Dhive.metastore.uris=thrift://metastore:9083" \
      --mount source=warehouse,target=/opt/hive/data/warehouse \
      --env IS_RESUME="true" \
      --name hiveserver2 aquabear/hive-ubi8:${HIVE_VERSION}
   ```
  
- HiveServer2, Metastore

To get a quick overview of both HiveServer2 and Metastore, there is a `docker-compose.yml` placed under `packaging/src/docker` for this purpose,
specify the `POSTGRES_LOCAL_PATH` first:
```shell
export POSTGRES_LOCAL_PATH=your_local_path_to_postgres_driver
```
Example:
```shell
mvn dependency:copy -Dartifact="org.postgresql:postgresql:42.7.3" && \
export POSTGRES_LOCAL_PATH=`mvn help:evaluate -Dexpression=settings.localRepository -q -DforceStdout`/org/postgresql/postgresql/42.7.3/postgresql-42.7.3.jar 
```
If you don't install maven or have problem in resolving the postgres driver, you can always download this jar yourself,
change the `POSTGRES_LOCAL_PATH` to the path of the downloaded jar.

Then,
```shell
docker compose up -d
```
HiveServer2, Metastore and Postgres services will be started as a consequence. 
Volumes are used to persist data generated by Hive inside Postgres and HiveServer2 containers,
  - hive_db

    The volume persists the metadata of Hive tables inside Postgres container.
  - warehouse

    The volume stores tables' files inside HiveServer2 container.

To stop/remove them all,
```shell
docker compose down
```

#### Usage

- HiveServer2 web
  - Accessed on browser at http://localhost:10002/
- Beeline:
  ```shell
   docker exec -it hiveserver2 beeline -u 'jdbc:hive2://hiveserver2:10000/'
   # If beeline is installed on host machine, HiveServer2 can be simply reached via:
   beeline -u 'jdbc:hive2://localhost:10000/'
  ```
- Run some queries
  ```sql
    show tables;
    create table hive_example(a string, b int) partitioned by(c int);
    alter table hive_example add partition(c=1);
    insert into hive_example partition(c=1) values('a', 1), ('a', 2),('b',3);
    select count(distinct a) from hive_example;
    select sum(b) from hive_example;
  ```
