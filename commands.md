# Project 2: Tracking User Activity

This document lays out the commands used to create my pipeline to get ready for data scientists who work for customers to run queries on the assessments data.


## Docker

This section outlines some commands for housekeeping and set-up with my docker containers.

```
# change directories
cd ~/w205/project-2-alisto92

# check stray containers
docker ps -a 

# remove 
docker rm -f xxxxx

# don't remove this one:
6793591eaeab        gcr.io/inverting-proxy/agent   "/bin/sh -c '/opt/bi…"   3 hours ago         Up 3 hours                              proxy-agent

# check network
docker network ls 

# remove all networks not used by at least one container (must type "y" to confirm)
docker network prune

# remove specific networks
docker network rm xxxx

# bring down docker images
docker pull midsw205/base:latest
docker pull midsw205/base:0.1.8
docker pull midsw205/base:0.1.9
docker pull redis
docker pull confluentinc/cp-zookeeper:latest
docker pull confluentinc/cp-kafka:latest
docker pull midsw205/spark-python:0.0.5
docker pull midsw205/spark-python:0.0.6
docker pull midsw205/cdh-minimal:latest
docker pull midsw205/hadoop:0.0.2
docker pull midsw205/presto:0.0.1

# bring docker cluster up 
docker-compose up -d

# pull image down -- when done working in container
docker-compose down

# check containers & network
docker ps -a
# if the above is working & the containers were spun up correctly, you should see the something similar to the following output:
CONTAINER ID        IMAGE                              COMMAND                  CREATED              STATUS        
      PORTS                                                                                NAMES
cb7eb59e21af        confluentinc/cp-kafka:latest       "/etc/confluent/dock…"   About a minute ago   Up About a min
ute   9092/tcp, 29092/tcp                                                                  project2alisto92_kafka_1
4759ac0c8406        midsw205/spark-python:0.0.5        "docker-entrypoint.s…"   About a minute ago   Up About a min
ute   0.0.0.0:8888->8888/tcp                                                               project2alisto92_spark_1
0d403238943d        confluentinc/cp-zookeeper:latest   "/etc/confluent/dock…"   About a minute ago   Up About a min
ute   2181/tcp, 2888/tcp, 3888/tcp, 32181/tcp                                              project2alisto92_zookeep
er_1
25419458f364        midsw205/cdh-minimal:latest        "cdh_startup_script.…"   About a minute ago   Up About a min
ute   8020/tcp, 8088/tcp, 8888/tcp, 9090/tcp, 11000/tcp, 11443/tcp, 19888/tcp, 50070/tcp   project2alisto92_clouder
a_1
8a9c69b82656        midsw205/base:latest               "/bin/bash"              About a minute ago   Up About a min
ute   8888/tcp                                                                             project2alisto92_mids_1
7041bf1d0472        gcr.io/inverting-proxy/agent       "/bin/sh -c '/opt/bi…"   22 minutes ago       Up 22 minutes 
                                                                                           proxy-agent
docker network ls
# if the above command is working & the containers were spun up correctly, you should see something similar to the following output:
NETWORK ID          NAME                       DRIVER              SCOPE
e42a1b76083e        bridge                     bridge              local
7faf4695f1bf        host                       host                local
5557405dd1c5        none                       null                local
55f604204654        project2alisto92_default   bridge              local

# check that hadoop is running correctly
docker-compose exec cloudera hadoop fs -ls /tmp/
# if it is, you should see the following output:
Found 1 items
drwxrwxrwt   - mapred mapred          0 2018-02-06 18:27 /tmp/hadoop-yarn

```


## Kafka

This section gives a set of commands for using Kafka to ingest the data.

```
# create a topic called "assessment"
docker-compose exec kafka kafka-topics --create --topic assessment --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181

# check topic
docker-compose exec kafka kafka-topics --describe --topic assessment --zookeeper zookeeper:32181

# expected output: 
Topic:assessment   PartitionCount:1    ReplicationFactor:1 Configs:
Topic: assessment  Partition: 0    Leader: 1    Replicas: 1  Isr: 1

# publish test messages
docker-compose exec mids bash -c "cat /w205/project-2-alisto92/assessment-attempts-20180128-121051-nested.json | jq '.[]' -c | kafkacat -P -b kafka:29092 -t assessments && echo 'Produced 100 messages.'"

# use kafkacat to produce test messages to the assessment topic
docker-compose exec mids bash -c "cat /w205/project-2-alisto92/assessment-attempts-20180128-121051-nested.json | jq '.[]' -c | kafkacat -P -b kafka:29092 -t assessment"

# check out messages -- WARNING: this will spur a printout of a large number of assessment data points and could take some time! 
# this prints the assessments in a format more machine-readable and hard to understand for a human:
docker-compose exec mids bash -c "cat /w205/project-2-alisto92/assessment-attempts-20180128-121051-nested.json"
# check out individual messages -- this is also more difficult for a human to read
docker-compose exec mids bash -c "cat /w205/project-2-alisto92/assessment-attempts-20180128-121051-nested.json | jq '.[]' -c"
# this prints the assessments in a format that is easier for a human to read:
docker-compose exec mids bash -c "cat /w205/project-2-alisto92/assessment-attempts-20180128-121051-nested.json | jq '.'"

# consume messages & print word count 
docker-compose exec mids bash -c "kafkacat -C -b kafka:29092 -t assessment -o beginning -e" | wc -l
```

## Pyspark using container

This section gives commands from using Spark to read the data from Kafka.

```
# run spark using spark container 
docker-compose exec spark pyspark

# read from kafka, at pyspark prompt
raw_assessments = spark.read.format("kafka").option("kafka.bootstrap.servers", "kafka:29092").option("subscribe","assessment").option("startingOffsets", "earliest").option("endingOffsets", "latest").load()

# see schema
raw_assessments.printSchema()

# see messages
raw_assessments.show()

# cache to reduce warnings
raw_assessments.cache()

# cast as strings
assessments = raw_assessments.select(raw_assessments.value.cast('string'))

# take a look 
assessments.show()
assessments.printSchema()
assessments.count()

# unrolling json
# pull out first entry 
assessments.select('value').take(1)
# pull out first entry and extract its value
assessments.select('value').take(1)[0].value

# use json to unroll 
import json

# pull out first message
first_assessment = json.loads(assessments.select('value').take(1)[0].value)

# take a look
first_assessment

# print an item from first message (the exam name)
print(first_assessment['exam_name'])

# write assessments in current form to hdfs
assessments.write.parquet("/tmp/assessments")

# check out results from another window
docker-compose exec cloudera hadoop fs -ls /tmp/
# expected output: 
Found 3 items
drwxr-xr-x   - root   supergroup          0 2020-03-05 20:54 /tmp/assessments
drwxrwxrwt   - mapred mapred              0 2018-02-06 18:27 /tmp/hadoop-yarn
drwx-wx-wx   - root   supergroup          0 2020-03-05 20:37 /tmp/hive

docker-compose exec cloudera hadoop fs -ls /tmp/assessments/
# expected output:
Found 2 items
-rw-r--r--   1 root supergroup          0 2020-03-05 20:54 /tmp/assessments/_SUCCESS
-rw-r--r--   1 root supergroup    2528606 2020-03-05 20:54 /tmp/assessments/part-00000-b3490376-7c8f-40ee-8288-e563
3e010d74-c000.snappy.parquet

# back in spark terminal window - What did we actually write?
assessments.show()
# expected output -- one can see that this is not a very helpful structure3
+--------------------+
|               value|
+--------------------+
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
|{"keen_timestamp"...|
+--------------------+
only showing top 20 rows

# use sys to deal with with unicode encoding
import sys
sys.stdout = open(sys.stdout.fileno(), mode='w', encoding='utf8', buffering=1)

# take a look at what we have using RDD -- to load the json 
import json
assessments.rdd.map(lambda x: json.loads(x.value)).toDF().show()

# unroll and save these extracted assessments -- using json 
extracted_assessments = assessments.rdd.map(lambda x: json.loads(x.value)).toDF()

# another way to unroll and save the extracted assessments -- not used here 
#from pyspark.sql import Row
#extracted_assessments = assessments.rdd.map(lambda x: Row(**json.loads(x.value))).toDF()

# take a look at unrolled version 
extracted_assessments.show()
# expected output -- one can see that the "sequences" column is still nested because it has "Map" in its value!
+--------------------+-------------+--------------------+------------------+--------------------+------------------
+------------+--------------------+--------------------+--------------------+
|        base_exam_id|certification|           exam_name|   keen_created_at|             keen_id|    keen_timestamp
|max_attempts|           sequences|          started_at|        user_exam_id|
+--------------------+-------------+--------------------+------------------+--------------------+------------------
+------------+--------------------+--------------------+--------------------+
|37f0a30a-7464-11e...|        false|Normal Forms and ...| 1516717442.735266|5a6745820eb8ab000...| 1516717442.735266
|         1.0|Map(questions -> ...|2018-01-23T14:23:...|6d4089e4-bde5-4a2...|
|37f0a30a-7464-11e...|        false|Normal Forms and ...| 1516717377.639827|5a674541ab6b0a000...| 1516717377.639827
|         1.0|Map(questions -> ...|2018-01-23T14:21:...|2fec1534-b41f-441...|
|4beeac16-bb83-4d5...|        false|The Principles of...| 1516738973.653394|5a67999d3ed3e3000...| 1516738973.653394
|         1.0|Map(questions -> ...|2018-01-23T20:22:...|8edbc8a8-4d26-429...|
|4beeac16-bb83-4d5...|        false|The Principles of...|1516738921.1137421|5a6799694fc7c7000...|1516738921.1137421
|         1.0|Map(questions -> ...|2018-01-23T20:21:...|c0ee680e-8892-4e6...|
|6442707e-7488-11e...|        false|Introduction to B...| 1516737000.212122|5a6791e824fccd000...| 1516737000.212122
|         1.0|Map(questions -> ...|2018-01-23T19:48:...|e4525b79-7904-405...|
|8b4488de-43a5-4ff...|        false|        Learning Git| 1516740790.309757|5a67a0b6852c2a000...| 1516740790.309757
|         1.0|Map(questions -> ...|2018-01-23T20:51:...|3186dafa-7acf-47e...|
|e1f07fac-5566-4fd...|        false|Git Fundamentals ...|1516746279.3801291|5a67b627cc80e6000...|1516746279.3801291
|         1.0|Map(questions -> ...|2018-01-23T22:24:...|48d88326-36a3-4cb...|
|7e2e0b53-a7ba-458...|        false|Introduction to P...| 1516743820.305464|5a67ac8cb0a5f4000...| 1516743820.305464
|         1.0|Map(questions -> ...|2018-01-23T21:43:...|bb152d6b-cada-41e...|
|1a233da8-e6e5-48a...|        false|Intermediate Pyth...|  1516743098.56811|5a67a9ba060087000...|  1516743098.56811
|         1.0|Map(questions -> ...|2018-01-23T21:31:...|70073d6f-ced5-4d0...|
|7e2e0b53-a7ba-458...|        false|Introduction to P...| 1516743764.813107|5a67ac54411aed000...| 1516743764.813107
|         1.0|Map(questions -> ...|2018-01-23T21:42:...|9eb6d4d6-fd1f-4f3...|
|4cdf9b5f-fdb7-4a4...|        false|A Practical Intro...|1516744091.3127241|5a67ad9b2ff312000...|1516744091.3127241
|         1.0|Map(questions -> ...|2018-01-23T21:45:...|093f1337-7090-457...|
|e1f07fac-5566-4fd...|        false|Git Fundamentals ...|1516746256.5878439|5a67b610baff90000...|1516746256.5878439
|         1.0|Map(questions -> ...|2018-01-23T22:24:...|0f576abb-958a-4c0...|
|87b4b3f9-3a86-435...|        false|Introduction to M...|  1516743832.99235|5a67ac9837b82b000...|  1516743832.99235
|         1.0|Map(questions -> ...|2018-01-23T21:40:...|0c18f48c-0018-450...|
|a7a65ec6-77dc-480...|        false|   Python Epiphanies|1516743332.7596769|5a67aaa4f21cc2000...|1516743332.7596769
|         1.0|Map(questions -> ...|2018-01-23T21:34:...|b38ac9d8-eef9-495...|
|7e2e0b53-a7ba-458...|        false|Introduction to P...| 1516743750.097306|5a67ac46f7bce8000...| 1516743750.097306
|         1.0|Map(questions -> ...|2018-01-23T21:41:...|bbc9865f-88ef-42e...|
|e5602ceb-6f0d-11e...|        false|Python Data Struc...|1516744410.4791961|5a67aedaf34e85000...|1516744410.4791961
|         1.0|Map(questions -> ...|2018-01-23T21:51:...|8a0266df-02d7-44e...|
|e5602ceb-6f0d-11e...|        false|Python Data Struc...|1516744446.3999851|5a67aefef5e149000...|1516744446.3999851
|         1.0|Map(questions -> ...|2018-01-23T21:53:...|95d4edb1-533f-445...|
|f432e2e3-7e3a-4a7...|        false|Working with Algo...| 1516744255.840405|5a67ae3f0c5f48000...| 1516744255.840405
|         1.0|Map(questions -> ...|2018-01-23T21:50:...|f9bc1eff-7e54-42a...|
|76a682de-6f0c-11e...|        false|Learning iPython ...| 1516744023.652257|5a67ad579d5057000...| 1516744023.652257
|         1.0|Map(questions -> ...|2018-01-23T21:46:...|dc4b35a7-399a-4bd...|
|a7a65ec6-77dc-480...|        false|   Python Epiphanies|1516743398.6451161|5a67aae6753fd6000...|1516743398.6451161
|         1.0|Map(questions -> ...|2018-01-23T21:35:...|d0f8249a-597e-4e1...|
+--------------------+-------------+--------------------+------------------+--------------------+------------------
+------------+--------------------+--------------------+--------------------+
only showing top 20 rows

# print the schema
extracted_assessments.printSchema()
# expected output -- we can confirm that "sequences" is nested
root
 |-- base_exam_id: string (nullable = true)
 |-- certification: string (nullable = true)
 |-- exam_name: string (nullable = true)
 |-- keen_created_at: string (nullable = true)
 |-- keen_id: string (nullable = true)
 |-- keen_timestamp: string (nullable = true)
 |-- max_attempts: string (nullable = true)
 |-- sequences: map (nullable = true)
 |    |-- key: string
 |    |-- value: array (valueContainsNull = true)
 |    |    |-- element: map (containsNull = true)
 |    |    |    |-- key: string
 |    |    |    |-- value: boolean (valueContainsNull = true)
 |-- started_at: string (nullable = true)
 |-- user_exam_id: string (nullable = true)


# save this as a parquet file
extracted_assessments.write.parquet("/tmp/extracted_assessments")

# check out new saved extracted file
docker-compose exec cloudera hadoop fs -ls /tmp/extracted_assessments/

# expected output:
Found 2 items
-rw-r--r--   1 root supergroup          0 2020-03-05 22:52 /tmp/extracted_assessments/_SUCCESS
-rw-r--r--   1 root supergroup     345388 2020-03-05 22:52 /tmp/extracted_assessments/part-00000-1da67e78-8d21-4e84
-b81d-41b937df9e25-c000.snappy.parquet

# exit pyspark
exit()
```

## Using Pyspark from Jupyter Notebook -- another option which was I did not use in implementing this project 

This section lays out commands for an alternative way to use PySpark (via notebook).

```
# exec a bash shell into spark container
docker-compose exec spark bash

# create symbolic link from spark directory to /w205
ln -s /w205 w205

# exit container
exit 

# start jupyter notebook for pyspark kernel
docker-compose exec spark env PYSPARK_DRIVER_PYTHON=jupyter PYSPARK_DRIVER_PYTHON_OPTS='notebook --no-browser --port 8888 --ip 0.0.0.0 --allow-root' pyspark

# copy the URL, and change IP address to the external IP address for Google cloud VM
# old http://35.247.73.16:8888/?token=e5552c283b72b0771d7b287baef4891b4bbcbba6e4045080
# old http://34.83.157.138:8888/?token=b246a0621090136fc73310fcc2dcaca72f760a639dafaf8f
http://34.83.157.138:8888/?token=63aa4ad2631a4f95b092481610894deb55371bb5793f57e8

# to access the notebook, open Google chrome browser incognito & use the above URL 

```


















