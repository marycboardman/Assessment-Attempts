# Assessment-Attempts
Data processing to track user activity incorporating docker containers, kafka, spark, and hadoop

# Project Overview

- Through 3 different activities, I spun up containers and prepared the infrastructure to land the data in the form and structure it needs to be to be queried. Specifically, I did the following: 
	- Published and consumed messages with kafka
	- Used spark to transform the messages
	- Landed the messages in hdfs.

## Introduction

To process this data, I logged into my Digital Ocean droplet, created a yml file, downloaded the data, and spun up a docker cluster with kafka, zookeeper, spark, hadoop, and the mids containers. I then published the messages and consumed them with kafka, consumed and printed data using spark, transformed and landed messages into hdfs, exited both spark and kafka, and spun down my docker cluster. 

## Data Preparation 

After logging into my droplet, I navigated to the currently existing w205 directory, and examined the files existing there.  

```
    cd ~/w205
    ls
```

Then, I created a new yml file as shown and annotated below: 

```
    vi docker-compose.yml
```
```
---
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest    # Zookeeper image most recently downloaded
    environment:
      ZOOKEEPER_CLIENT_PORT: 32181
      ZOOKEEPER_TICK_TIME: 2000
    expose:
      - "2181"
      - "2888"
      - "32181"
      - "3888"

  kafka:
    image: confluentinc/cp-kafka:latest    # Kafka image most recently downloaded
    depends_on:
      - zookeeper
    environment:    # Configure the broker ID, connect to zookeeper, and also set the listener configuration 
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:32181    # Connects to the zookeeper port
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    expose:
      - "9092"
      - "29092"    # internal connection

  cloudera:
    image: midsw205/cdh-minimal:latest    # Cloudera image most recently downloaded to run the hadoop cluster
    expose:
      - "8020" # nn
      - "50070" # nn http
      - "8888" # hue
    #ports:
    #- "8888:8888" # Ports used to connect

  spark:
    image: midsw205/spark-python:0.0.5    # Spark image most recently downloaded
    stdin_open: true
    tty: true
    volumes:
      - ~/w205:/w205    # File where this is run
    command: bash    # Spark to be run on a bash shell
    depends_on:
      - cloudera
    environment:
      HADOOP_NAMENODE: cloudera   

  mids:
    image: midsw205/base:latest    # MIDSw205 image most recently downloaded. 
    stdin_open: true
    tty: true
    volumes:
      - ~/w205:/w205    # File where this is run

```

Then, I downloaded the json file to my w205 folder and checked to make sure it was there. 
```
    curl -L -o assessment-attempts-20180128-121051-nested.json https://goo.gl/f5bRm4
    ls
```

I spun up the docker cluster, and started the kafka logs. Then, I switched to a  different terminal window to continue.

```
    docker-compose up -d
    docker-compose logs -f kafka
```

Once my cluster was spun up and running, I ran the following code to view the hadoop hdfs file system, since it is separate from the local linux file system in my droplet. 

```
docker-compose exec cloudera hadoop fs -ls /tmp/
```

Below is the output, which showed one file:
```
Found 1 items
drwxrwxrwt   - mapred mapred          0 2018-02-06 18:27 /tmp/hadoop-yarn
```

I then created a topic, "players" in the kafka container. This code also shows that I created 1 partition, had a replication factor of 1, and am using zookeeper to manage the process. I used the kafka-topics utility to do this. 

```
docker-compose exec kafka \
  kafka-topics \
    --create \
    --topic players \
    --partitions 1 \
    --replication-factor 1 \
    --if-not-exists \
    --zookeeper zookeeper:32181
```

My output was the following, as expected:

```
Created topic "players".
```

Then, I used the kafka-topics utility to check the topic in the kafka container.

```
docker-compose exec kafka \
  kafka-topics \
  --describe \
  --topic players \
  --zookeeper zookeeper:32181
```

My output was the following, also as expected:

```
Topic:players	PartitionCount:1	ReplicationFactor:1	Configs:
	Topic: players	Partition: 0	Leader: 1	Replicas: 1	Isr: 1
```

Since it matched my configuration, it was a sign that everything was up and running as it should be. Then, as prescribed in the assignment, I ran the docker-compose exec command. What this command does is read in the json file previously downloaded and used the kafkacat utility to publish 100 messages to the players topic in kafka. 

```
docker-compose exec mids \
  bash -c "cat /w205/assessment-attempts-20180128-121051-nested.json | jq '.[]' \
    -c | kafkacat -P -b kafka:29092 \
    -t players"

```
Since the code didn't specify output, there was no output, as expected.  

To recreate what I did previously, I then consumed 100 messages from the players topic. I did this using the kafka-console-consumer utility. I read them in from the beginning. 

```
docker-compose exec kafka \
  kafka-console-consumer \
    --bootstrap-server kafka:29092 \
    --topic players \
    --from-beginning 
    --max-messages 100
```

There was a long output that was json (but without the readable formatting), and then the shorter output of the following: 

```
Processed a total of 100 messages.
```

Then, I determined the word count using the following code: 

```
docker-compose exec mids \
  bash -c "kafkacat -C -b kafka:29092 \
    -t players -o beginning -e" | wc -l
```

The word count was 3281. 

To run Spark, I then switched terminal windows to run my spark container.

```
docker-compose exec spark pyspark
```

The first thing I did was to consume messages from the kafka topic "players", and create a spark data frame called "raw_players". 

```
df = spark \
  .read \
  .format("kafka") \
  .option("kafka.bootstrap.servers", "kafka:29092") \
  .option("subscribe","foo") \
  .option("startingOffsets", "earliest") \
  .option("endingOffsets", "latest") \
  .load() 
```

Then, I used the following command to cache the data structure. This way, it doesn't print the warning messages that aren't useful. Below is the command with the output. 

```
raw_players.cache()
DataFrame[key: binary, value: binary, topic: string, partition: int, offset: bigint, timestamp: timestamp, timestampType: int]
```

I then printed the schema using the following code, and seeing the following output:

```
raw_players.printSchema()
root
 |-- key: binary (nullable = true)
 |-- value: binary (nullable = true)
 |-- topic: string (nullable = true)
 |-- partition: integer (nullable = true)
 |-- offset: long (nullable = true)
 |-- timestamp: timestamp (nullable = true)
 |-- timestampType: integer (nullable = true)
```

I used the following code to cast the keys and values from binary to string. Then, I printed the schema to check my work. 

```
players = raw_players.select(raw_players.value.cast('string'))
players.printSchema()
root
 |-- value: string (nullable = true)
```
Then, I wrote the players data frame into a parquet file in the hadoop hfs. This is a binary format and immutable. 

```
players.write.parquet("/tmp/players")
```
I then switched back to my docker command line window and used the following commands to view the directory. Below we can see players as a directory and not a file. 

```
docker-compose exec cloudera hadoop fs -ls /tmp/
Found 3 items
drwxr-xr-x   - root   supergroup          0 2018-07-09 00:17 /tmp/commits
drwxrwxrwt   - mapred mapred              0 2018-02-06 18:27 /tmp/hadoop-yarn
drwx-wx-wx   - root   supergroup          0 2018-07-09 00:14 /tmp/hive

docker-compose exec cloudera hadoop fs -ls /tmp/players/
Found 2 items
-rw-r--r--   1 root supergroup          0 2018-07-09 02:17 /tmp/players/_SUCCESS
-rw-r--r--   1 root supergroup     406096 2018-07-09 02:17 /tmp/players/part-00000-98c643a4-d107-4843-97a1-c60d61851270-c000.snappy.parquet
```

Then, I went back to my spark container and printed the top 20 rows of the players data frame. 

```
players.show()
+--------------------+
|               value|
+--------------------+
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
+--------------------+
only showing top 20 rows
```
I imported the sys package and then set a standard output that writes data to utf-8 instead of unicode. 

```
import sys
sys.stdout = open(sys.stdout.fileno(), mode='w', encoding='utf8', buffering=1)
```

Because the output was difficult to read and decipher, I decided to import and use the json package, printing out some of the data. Below is the code with the following output (and warning message). 

```
import json
players.rdd.map(lambda x: json.loads(x.value)).toDF().show()
/spark-2.2.0-bin-hadoop2.6/python/pyspark/sql/session.py:351: UserWarning: Using RDD of dict to inferSchema is deprecated. Use pyspark.sql.Row instead
  warnings.warn("Using RDD of dict to inferSchema is deprecated. "

+--------------------+-----------+-----------+-----------+--------------------+---------+------+--------+---------+----+
|                Club|ClubCountry|Competition|DateOfBirth|            FullName|IsCaptain|Number|Position|     Team|Year|
+--------------------+-----------+-----------+-----------+--------------------+---------+------+--------+---------+----+
|Club AtlÃ©tico Ta...|  Argentina|  World Cup|   1905-5-5|        Ãngel Bossio|    false|      |      GK|Argentina|1930|
|Quilmes AtlÃ©tico...|  Argentina|  World Cup| 1908-10-23|        Juan Botasso|    false|      |      GK|Argentina|1930|
|          Boca Junio|  Argentina|  World Cup|  1907-2-23|      Roberto Cherro|    false|      |      FW|Argentina|1930|
|Central Norte TucumÃ|  Argentina|  World Cup|  1907-2-23|   Alberto Chividini|    false|      |      DF|Argentina|1930|
|Club Atletico Est...|  Argentina|  World Cup|  1909-3-19|                    |    false|    10|      FW|Argentina|1930|
|Racing Club de Av...|  Argentina|  World Cup|  1906-3-23|                    |    false|      |      DF|Argentina|1930|
|     Sportivo Barrac|  Argentina|  World Cup|  1902-6-20|       Juan Evaristo|    false|      |      DF|Argentina|1930|
|          Boca Junio|  Argentina|  World Cup| 1908-12-10|      Mario Evaristo|    false|      |      FW|Argentina|1930|
|Estudiantes de La...|  Argentina|  World Cup| 1905-10-22|     Manuel Ferreira|     true|      |      FW|Argentina|1930|
|Club AtlÃ©tico Sa...|  Argentina|  World Cup|  1901-5-15|          Luis Monti|    false|      |      MF|Argentina|1930|
|          Boca Junio|  Argentina|  World Cup|  1899-3-12|                    |    false|      |      DF|Argentina|1930|
|Club Atletico Est...|  Argentina|  World Cup|   1905-1-1|   Rodolfo Orlandini|    false|      |      DF|Argentina|1930|
|Racing Club de Av...|  Argentina|  World Cup|  1903-5-24|Fernando Paternoster|    false|      |      DF|Argentina|1930|
|Racing Club de Av...|  Argentina|  World Cup| 1900-12-28|   Natalio Perinetti|    false|      |      FW|Argentina|1930|
|Club Sportivo Bue...|  Argentina|  World Cup|  1908-9-13|     Carlos Peucelle|    false|      |      FW|Argentina|1930|
|Club AtlÃ©tico La...|  Argentina|  World Cup|  1910-10-3|     Edmundo Piaggio|    false|      |      DF|Argentina|1930|
|Estudiantes de La...|  Argentina|  World Cup|  1908-5-12|  Alejandro Scopelli|    false|      |      FW|Argentina|1930|
|Club AtlÃ©tico La...|  Argentina|  World Cup|   1902-2-5|      Carlos Spadaro|    false|    11|      FW|Argentina|1930|
|Club AtlÃ©tico Hu...|  Argentina|  World Cup|  1905-1-17|                    |    false|      |      FW|Argentina|1930|
|          Boca Junio|  Argentina|  World Cup| 1909-12-21|                    |    false|      |      MF|Argentina|1930|
+--------------------+-----------+-----------+-----------+--------------------+---------+------+--------+---------+----+
only showing top 20 rows
```
I then created a new data frame to hold the data in json, and then printed off the first 20 rows. Below is the code with the following output (and warning message). 

```
extracted_players = players.rdd.map(lambda x: json.loads(x.value)).toDF()
extracted_players.show()
+--------------------+-----------+-----------+-----------+--------------------+---------+------+--------+---------+----+
|                Club|ClubCountry|Competition|DateOfBirth|            FullName|IsCaptain|Number|Position|     Team|Year|
+--------------------+-----------+-----------+-----------+--------------------+---------+------+--------+---------+----+
|Club AtlÃ©tico Ta...|  Argentina|  World Cup|   1905-5-5|        Ãngel Bossio|    false|      |      GK|Argentina|1930|
|Quilmes AtlÃ©tico...|  Argentina|  World Cup| 1908-10-23|        Juan Botasso|    false|      |      GK|Argentina|1930|
|          Boca Junio|  Argentina|  World Cup|  1907-2-23|      Roberto Cherro|    false|      |      FW|Argentina|1930|
|Central Norte TucumÃ|  Argentina|  World Cup|  1907-2-23|   Alberto Chividini|    false|      |      DF|Argentina|1930|
|Club Atletico Est...|  Argentina|  World Cup|  1909-3-19|                    |    false|    10|      FW|Argentina|1930|
|Racing Club de Av...|  Argentina|  World Cup|  1906-3-23|                    |    false|      |      DF|Argentina|1930|
|     Sportivo Barrac|  Argentina|  World Cup|  1902-6-20|       Juan Evaristo|    false|      |      DF|Argentina|1930|
|          Boca Junio|  Argentina|  World Cup| 1908-12-10|      Mario Evaristo|    false|      |      FW|Argentina|1930|
|Estudiantes de La...|  Argentina|  World Cup| 1905-10-22|     Manuel Ferreira|     true|      |      FW|Argentina|1930|
|Club AtlÃ©tico Sa...|  Argentina|  World Cup|  1901-5-15|          Luis Monti|    false|      |      MF|Argentina|1930|
|          Boca Junio|  Argentina|  World Cup|  1899-3-12|                    |    false|      |      DF|Argentina|1930|
|Club Atletico Est...|  Argentina|  World Cup|   1905-1-1|   Rodolfo Orlandini|    false|      |      DF|Argentina|1930|
|Racing Club de Av...|  Argentina|  World Cup|  1903-5-24|Fernando Paternoster|    false|      |      DF|Argentina|1930|
|Racing Club de Av...|  Argentina|  World Cup| 1900-12-28|   Natalio Perinetti|    false|      |      FW|Argentina|1930|
|Club Sportivo Bue...|  Argentina|  World Cup|  1908-9-13|     Carlos Peucelle|    false|      |      FW|Argentina|1930|
|Club AtlÃ©tico La...|  Argentina|  World Cup|  1910-10-3|     Edmundo Piaggio|    false|      |      DF|Argentina|1930|
|Estudiantes de La...|  Argentina|  World Cup|  1908-5-12|  Alejandro Scopelli|    false|      |      FW|Argentina|1930|
|Club AtlÃ©tico La...|  Argentina|  World Cup|   1902-2-5|      Carlos Spadaro|    false|    11|      FW|Argentina|1930|
|Club AtlÃ©tico Hu...|  Argentina|  World Cup|  1905-1-17|                    |    false|      |      FW|Argentina|1930|
|          Boca Junio|  Argentina|  World Cup| 1909-12-21|                    |    false|      |      MF|Argentina|1930|
+--------------------+-----------+-----------+-----------+--------------------+---------+------+--------+---------+----+
only showing top 20 rows
```
Then, I saved extracted_players to a parquet file in hdfs. For comparison, I then printed off both players and extracted_players. Below is the code and the output:
```
extracted_players.write.parquet("/tmp/extracted_players")
players.show()
+--------------------+
|               value|
+--------------------+
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
|{"Competition":"W...|
+--------------------+
only showing top 20 rows
```
```
extracted_players.show()
+--------------------+-----------+-----------+-----------+--------------------+---------+------+--------+---------+----+
|                Club|ClubCountry|Competition|DateOfBirth|            FullName|IsCaptain|Number|Position|     Team|Year|
+--------------------+-----------+-----------+-----------+--------------------+---------+------+--------+---------+----+
|Club AtlÃ©tico Ta...|  Argentina|  World Cup|   1905-5-5|        Ãngel Bossio|    false|      |      GK|Argentina|1930|
|Quilmes AtlÃ©tico...|  Argentina|  World Cup| 1908-10-23|        Juan Botasso|    false|      |      GK|Argentina|1930|
|          Boca Junio|  Argentina|  World Cup|  1907-2-23|      Roberto Cherro|    false|      |      FW|Argentina|1930|
|Central Norte TucumÃ|  Argentina|  World Cup|  1907-2-23|   Alberto Chividini|    false|      |      DF|Argentina|1930|
|Club Atletico Est...|  Argentina|  World Cup|  1909-3-19|                    |    false|    10|      FW|Argentina|1930|
|Racing Club de Av...|  Argentina|  World Cup|  1906-3-23|                    |    false|      |      DF|Argentina|1930|
|     Sportivo Barrac|  Argentina|  World Cup|  1902-6-20|       Juan Evaristo|    false|      |      DF|Argentina|1930|
|          Boca Junio|  Argentina|  World Cup| 1908-12-10|      Mario Evaristo|    false|      |      FW|Argentina|1930|
|Estudiantes de La...|  Argentina|  World Cup| 1905-10-22|     Manuel Ferreira|     true|      |      FW|Argentina|1930|
|Club AtlÃ©tico Sa...|  Argentina|  World Cup|  1901-5-15|          Luis Monti|    false|      |      MF|Argentina|1930|
|          Boca Junio|  Argentina|  World Cup|  1899-3-12|                    |    false|      |      DF|Argentina|1930|
|Club Atletico Est...|  Argentina|  World Cup|   1905-1-1|   Rodolfo Orlandini|    false|      |      DF|Argentina|1930|
|Racing Club de Av...|  Argentina|  World Cup|  1903-5-24|Fernando Paternoster|    false|      |      DF|Argentina|1930|
|Racing Club de Av...|  Argentina|  World Cup| 1900-12-28|   Natalio Perinetti|    false|      |      FW|Argentina|1930|
|Club Sportivo Bue...|  Argentina|  World Cup|  1908-9-13|     Carlos Peucelle|    false|      |      FW|Argentina|1930|
|Club AtlÃ©tico La...|  Argentina|  World Cup|  1910-10-3|     Edmundo Piaggio|    false|      |      DF|Argentina|1930|
|Estudiantes de La...|  Argentina|  World Cup|  1908-5-12|  Alejandro Scopelli|    false|      |      FW|Argentina|1930|
|Club AtlÃ©tico La...|  Argentina|  World Cup|   1902-2-5|      Carlos Spadaro|    false|    11|      FW|Argentina|1930|
|Club AtlÃ©tico Hu...|  Argentina|  World Cup|  1905-1-17|                    |    false|      |      FW|Argentina|1930|
|          Boca Junio|  Argentina|  World Cup| 1909-12-21|                    |    false|      |      MF|Argentina|1930|
+--------------------+-----------+-----------+-----------+--------------------+---------+------+--------+---------+----+
only showing top 20 rows
```
I then created a second kafka topics in my docker window. This one was called "commits" and existed alongside "players". I used kafkacat to publish the json data to the commits topic. 

```
docker-compose exec kafka \
  kafka-topics \
    --create \
    --topic commits \
    --partitions 1 \
    --replication-factor 1 \
    --if-not-exists \
    --zookeeper zookeeper:32181
```
```
docker-compose exec mids \
  bash -c "cat /w205/spark-with-kafka-and-hdfs/github-example-large.json \
    | jq '.[]' -c \
    | kafkacat -P -b kafka:29092 -t commits"
```
Going back to my spark container, I consumed data from the commits kafka topic into a new data frame called "raw_commits".

```
raw_commits = spark \
  .read \
  .format("kafka") \
  .option("kafka.bootstrap.servers", "kafka:29092") \
  .option("subscribe","commits") \
  .option("startingOffsets", "earliest") \
  .option("endingOffsets", "latest") \
  .load() 
```
Just like before, I ran the cache command, printed the schema, and cast the binary value data to string data. 

```
raw_commits.cache()
DataFrame[key: binary, value: binary, topic: string, partition: int, offset: bigint, timestamp: timestamp, timestampType: int]
```
```
raw_commits.printSchema()
root
 |-- key: binary (nullable = true)
 |-- value: binary (nullable = true)
 |-- topic: string (nullable = true)
 |-- partition: integer (nullable = true)
 |-- offset: long (nullable = true)
 |-- timestamp: timestamp (nullable = true)
 |-- timestampType: integer (nullable = true)
```
commits = raw_commits.select(raw_commits.value.cast('string'))
```
The output reflected the output from the players topic. Given the data is the same, this was expected. Just as with the players topic, I then wrote the data frame to a parquet file into hdfs. 
```
commits.write.parquet("/tmp/commits")
```
I extracted the json fields, and printed the data frame. Interestingly, the json data is nested, when it was flat before. (This also makes it more difficult to read.)
```
extracted_commits = commits.rdd.map(lambda x: json.loads(x.value)).toDF()
extracted_commits.show()
+--------------------+--------------------+--------------------+--------------------+--------------------+--------------------+--------------------+--------------------+
|              author|        comments_url|              commit|           committer|            html_url|             parents|                 sha|                 url|
+--------------------+--------------------+--------------------+--------------------+--------------------+--------------------+--------------------+--------------------+
|                null|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> 61edf...|bd34b8dd2e4414409...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> 8eff7...|61edf3fa93f6177ef...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> b4742...|8eff744eecb9ab2f4...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> 9a457...|b4742c12570481786...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> b5560...|9a4576e7567dd38b9...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> 79ece...|b5560d8420d330c4f...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> f06de...|79ece359819cdd7d0...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> 0c9ea...|f06deb828a318536b...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> 476b3...|0c9eacedaae1e0d53...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> 23927...|476b36770d9337381...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> 98b36...|239278fd3a02dc1ae...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> c3cbb...|98b36e74b8174da66...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> 18879...|c3cbbdd8a201aafdc...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> bdadd...|18879fb99367924cd...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> 7b81a...|bdaddcf10730e2a26...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> c5382...|7b81a836c31500e68...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> 4a624...|c538237f4e4c381d3...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> 1900c...|4a6241be0697bbe4e...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> 578d5...|1900c7bcac7677778...|https://api.githu...|
|Map(gists_url -> ...|https://api.githu...|Map(author -> Map...|Map(gists_url -> ...|https://github.co...|[Map(sha -> b0d6d...|578d536233b628847...|https://api.githu...|
+--------------------+--------------------+--------------------+--------------------+--------------------+--------------------+--------------------+--------------------+
only showing top 20 rows
```
Then, I printed the schema. It shows clearly the nested json data. 
```
extracted_commits.printSchema()
root
 |-- author: map (nullable = true)
 |    |-- key: string
 |    |-- value: string (valueContainsNull = true)
 |-- comments_url: string (nullable = true)
 |-- commit: map (nullable = true)
 |    |-- key: string
 |    |-- value: map (valueContainsNull = true)
 |    |    |-- key: string
 |    |    |-- value: string (valueContainsNull = true)
 |-- committer: map (nullable = true)
 |    |-- key: string
 |    |-- value: string (valueContainsNull = true)
 |-- html_url: string (nullable = true)
 |-- parents: array (nullable = true)
 |    |-- element: map (containsNull = true)
 |    |    |-- key: string
 |    |    |-- value: string (valueContainsNull = true)
 |-- sha: string (nullable = true)
 |-- url: string (nullable = true)
```
To work with this, I used spark sql to work with this nested json. I created a temporary table based on the data frame, called "commits". Then, I queried spark sql against the temporary table created. 

```
extracted_commits.registerTempTable('commits')
spark.sql("select commit.committer.name from commits limit 10").show()
+----------------+
|            name|
+----------------+
|   Nico Williams|
|   Nico Williams|
|Nicolas Williams|
|Nicolas Williams|
|Nicolas Williams|
|Nicolas Williams|
|William Langford|
|William Langford|
|William Langford|
|   Nico Williams|
+----------------+
```
```
spark.sql("select commit.committer.name, commit.committer.date, sha from commits limit 10").show()
+----------------+--------------------+--------------------+
|            name|                date|                 sha|
+----------------+--------------------+--------------------+
|   Nico Williams|2018-01-04T21:50:20Z|bd34b8dd2e4414409...|
|   Nico Williams|2017-12-13T00:27:59Z|61edf3fa93f6177ef...|
|Nicolas Williams|2017-12-11T18:22:02Z|8eff744eecb9ab2f4...|
|Nicolas Williams|2017-12-11T18:17:17Z|b4742c12570481786...|
|Nicolas Williams|2017-12-11T17:20:16Z|9a4576e7567dd38b9...|
|Nicolas Williams|2017-12-11T05:09:35Z|b5560d8420d330c4f...|
|William Langford|2017-12-05T01:10:56Z|79ece359819cdd7d0...|
|William Langford|2017-12-05T00:20:58Z|f06deb828a318536b...|
|William Langford|2017-11-30T01:47:56Z|0c9eacedaae1e0d53...|
|   Nico Williams|2017-11-29T17:46:08Z|476b36770d9337381...|
+----------------+--------------------+--------------------+
```
Finally, saved the query results to a data frame, and wrote that data frame to a parquet file in hdfs before exiting spark. 
```
some_commit_info = spark.sql("select commit.committer.name, commit.committer.date, sha from commits limit 10")
some_commit_info.write.parquet("/tmp/some_commit_info")
```
Back in my docker container, I ran the following commands to see the hdfs directory and files created.
```
docker-compose exec cloudera hadoop fs -ls /tmp/
Found 6 items
drwxr-xr-x   - root   supergroup          0 2018-07-09 00:17 /tmp/commits
drwxr-xr-x   - root   supergroup          0 2018-07-09 02:21 /tmp/extracted_players
drwxrwxrwt   - mapred mapred              0 2018-02-06 18:27 /tmp/hadoop-yarn
drwx-wx-wx   - root   supergroup          0 2018-07-09 00:14 /tmp/hive
drwxr-xr-x   - root   supergroup          0 2018-07-09 02:17 /tmp/players
drwxr-xr-x   - root   supergroup          0 2018-07-09 03:29 /tmp/some_commit_info
```
```
docker-compose exec cloudera hadoop fs -ls /tmp/commits/
Found 2 items
-rw-r--r--   1 root supergroup          0 2018-07-09 00:17 /tmp/commits/_SUCCESS
-rw-r--r--   1 root supergroup        266 2018-07-09 00:17 /tmp/commits/part-00000-997f3bbb-8cae-49e9-981c-90cffd3f1cb0-c000.snappy.parquet
```
To close up, I used the exit() command to exit my spark container, and ^C to exit my kafka logs. I closed connections with both of those windows and I spun down my docker cluster, then checked to make sure there were no docker containers running. 

    docker-compose down
    docker-compose ps
    docker ps -a

## Summary

In this project, I demonstrated how to publish and consume messages with kafka, consume and print data with spark. I did this by spinning up a cluster with kafka, zookeeper, spark, hadoop, and the mids container. To do this, I downloaded the messages and replaced the yml file. Finally, in addition to the stated above, I demonstrated in spark how to transform data, work with nested json structures, and land messages into parquet files into the hdfs directory. I also created two kafka topics and had them running simultaneously. 

