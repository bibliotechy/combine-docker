# Combine-Docker


## Overview

This repository provides a "Dockerized" version of [Combine](https://github.com/wsulib/combine.git).

### Hoes does it work?

Major components that support Combine, all installed on a single server when building via the [Combine-Playbook](https://github.com/wsulib/combine-playbook.git) ansible route, have been broken out into distinct Docker images and containers.  Using [Docker Compose](https://docs.docker.com/compose/), each of these major components is associated with a Docker Compose **service**.  Some share base images, others are pulled from 3rd party Docker images (like ElasticSearch and Mongo).

Docker Compose provides a way to interact with all the containers that support Combine at once, even providing some improved ability to view logs, restart services, etc.

### Data Integrity

**WARNING:** The containerization of Combine provides arguably easier deployment and upgrading, but introduces some risks to data integrity in Combine.  Combine-Docker stores data that needs to persist between container rebuilding and upgrades in named volumes, specifically the following:

  * `esdata`: ElasticSearch data
  * `mongodata`: Mongo data
  * `mysqldata`: MySQL data
  * `hdfs`: Hadoop HDFS data
  * `combine_home`: Home directory for Combine that contains important, non-volatile data

Containers are shutdown with the command `docker-compose down`, which is perfectly safe and encouraged!  However, **do not** include the `-v` or `--volumes` flag to that command, which will remove **all** volumes associated with the services in the `docker-compose.yml` file.

Docker does a relatively good job protecting named volumes, but this simple command would wipe data from Combine.  You can find [more information about the command `docker-compose down` here](https://docs.docker.com/compose/reference/down/).


## Installation and First Build

The first step is to clone this repository and move into it:
```
git clone https://github.com/wsulib/combine-docker.git
cd combine-docker
```

Next, run the `first_build.sh` script:
```
./first_build.sh
```

**Note:** This script may take some time, anywhere from 5-20 minutes depending on your hardware.  This script accomplishes a few things:

  * initializes Combine Django app as Git submodule at `./combine/combine`
  * builds all required docker images
  * runs one-time database initializations and migrations


## Configuration

Once a build is complete, configurations may be performed on Combine's `localsettings.py`.  This file is found at `./combine/combine/combine/localsettings.py`.  This file will be maintained between upgrades.


## Running and Managing

Ensuring that `first_build.sh` (or `update_build.sh` if appropriate) has been run, fire up all containers with the following:
```
docker-compose up -d
```

Logs can be viewed with the `logs` command, again, selecting all services, or a subset of:
```
# tail all logs
docker-compose logs -f

# tail logs of specific services
docker-compose logs -f combine-django combine-celery livy

```

As outlined in the [Combine-Docker Containers](#docker-images-and-containers) section all services, or a subset of, can be restarted as follows:
```
# e.g. restart Combine Django app, background tasks Celery, and Livy
docker-compose restart combine-django combine-celery

# e.e. restart everything
docker-compose restart
```

To stop all services and containers (**NOTE:** Do not include `-v` or `--volumes` flags, as these will wipe ALL data from Combine):
```
docker-compose down
```

View stats of containers:
```
docker stats
```


## Updating

This dockerized version of Combine supports, arguably, easier version updating becaues major components, broken out as images and containers, can be readily rebuilt.  Much like the repository Combine-Playbook, this repository follows the same versioning as Combine.  So checking out the tagged release `v0.7` for this repository, will build Combine version `v0.7`.

To update, follow these steps from the Combine-Docker repository root folder:

```
# pull new changes to Combine-Docker repository
git pull

# checkout desired release, e.g. v0.7
git checkout v0.7

# run update build script
./update_build.sh

# Restart as per normal
docker-compose up -d
```


## Docker Services and Volumes & Binds

This dockerized version of Combine includes the following services, where each becomes a single container:

| Service Name          | Internal Network IP | Notes                                                      |
| --------------------- | ------------------- | ---------------------------------------------------------- |
| **host machine**      | `10.5.0.1`          | not a container, but part of internal network              |
| `elasticsearch`       | `10.5.0.2`          |                                                            |
| `mongo`               | `10.5.0.3`          |                                                            |
| `mysql`               | `10.5.0.4`          |                                                            |
| `redis`               | `10.5.0.5`          |                                                            |
| `hadoop-namenode`     | `10.5.0.6`          |                                                            |
| `hadoop-datanode`     | `10.5.0.7`          |                                                            |
| `spark-master`        | `10.5.0.8`          | not currently used                                         |
| `spark-worker`        | `10.5.0.9`          | not currently used                                         |
| `combine-django`      | `10.5.0.10`         |                                                            |
| `livy`                | `10.5.0.11`         | location of spark application running in `local[*]` mode   |
| `combine-celery`      | `10.5.0.12`         |                                                            |


The following tables show Docker volumes and binds that are created to support data sharing between containers, and "long-term" data storage.  The column `Data Storage` indicates which volumes act as data stores for Combine and should not be deleted (unless, of course, a fresh installation is desired).  Conversely, the column `Refreshed on Upgrade` shows which tables are removed during builds.  **Note:** this information is purely for informational purposes only; the build scripts and normal usage of `docker-compose up` and `docker-compose down` will not remove these volumes.


|Volume Name|Type|Source|Target|Data Storage|RefreshedonUpdate|AssociatedServices|
|-----------|----|------|------|------------|-----------------|------------------|
|`esdata`|namedvolume|n/a|`/usr/share/elasticsearch/data`|TRUE||elasticsearch|
|`mongodata`|namedvolume|n/a|`/data/db`|TRUE||mongo|
|`mysqldata`|namedvolume|n/a|`/var/lib/mysql`|TRUE||mysql|
|`hdfs`|namedvolume|n/a|`/hdfs`|TRUE||hadoop-namenode,hadoop-datanode|
|`combine_home`|namedvolume|n/a|`/home/combine`|TRUE||[spark-cluster-base]|
|`combine_django_app`|bind|`./combine/combine`|`/opt/combine`||TRUE|combine-django,combine-celery,livy|
|`combine_python_env`|namedvolume|n/a|`/opt/conda/envs/combine`||TRUE|combine-django,combine-celery,livy|
|`hadoop_binaries`|namedvolume|n/a|`/opt/hadoop`||TRUE|[spark-cluster-base]|
|`spark_binaries`|namedvolume|n/a|`/opt/spark`||TRUE|[spark-cluster-base]|
|`livy_binaries`|namedvolume|n/a|`/opt/livy`||TRUE|[spark-cluster-base]|
|`combine_tmp`|namedvolume|n/a|`/tmp`||TRUE|[spark-cluster-base]|


## Troubleshooting

### ElasticSearch container dies because of `vm.max_map_count`

Depending on machine and OS (Linux, Mac, Windows), might need to bump `vm.max_map_count` on Docker host machine (seems to be particulary true on older ones):
[https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-cli-run-prod-mode](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-cli-run-prod-mode)

### Port collision error: `port is already allocated`

By default, nearly all relevant ports are exposed from the containers that conspire to run Combine, but these can turned off selectively (or changed) if you have services running on your host that conflict.  Look for the `ports` section for each service in the `docker-compose.yml` to enable or disable them.

### Other issues?

Please don't hesitate to [submit an issue](https://github.com/WSULib/combine-docker/issues)!


## Development

The Combine Django application, where most developments efforts are targeted, is a [bind mount volume](https://docs.docker.com/storage/bind-mounts/) from the location of this cloned repository on disk at `./combine/combine`.  Though the application is copied to the docker images during build, to support the installation of dependencies, the location `/opt/combine` is overwritten by this bind volume at `docker-compose up` or `run`.  This allows live editing of the local folder `./combine/combine`, which is updating the folder `/opt/combine` in services `combine-django`, `combine-celery`, and `livy`.

The folder `./combine/combine` can, for the most part, be treated like a normal GitHub repository.  For example, one could checkout or create a new branch, and then push and pull from there.



