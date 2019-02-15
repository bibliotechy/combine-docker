# TODO

## Combine Django app

  * checking supervisor in multiple places, hardcoded to `None`, need to update
  * when starting, or running initial migrations, wait for MySQL to be ready
    * ping port `3306`?

## Celery

  * handle checking of status without use of supervisor (and in fact, should support both)

## Mongo

  * migrate all `localhost` and `127.0.0.1` to `settings.MONGO_HOST`

## ElasticSearch

  * configure elasticsearch host, same story as Mongo

## Initial Build

  * create indices for Mongo collections on build:
  `https://github.com/WSULib/combine-playbook/blob/master/roles/combine/tasks/main.yml#L98`
  * format HDFS node:
  `docker-compose run hadoop-namenode /bin/bash -c "mkdir -p /hdfs/namenode && echo 'Y' | ${HADOOP_PREFIX}/bin/hdfs namenode -format"`

## Misc

  * determine approach for es2csv and ElasticDump, both of which exist on OS
    * es2csv uses python 2.7, maybe own image?  both together in dedicated container/image?