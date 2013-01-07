# Intro
Camus is LinkedIn's [Kafka](http://kafka.apache.org "Kafka")->HDFS pipeline. It is a mapreduce job that does distributed data loads out of Kafka. It includes the following features:

* Automatic discovery of topics
* Schema management
* Date partitioning

It is used at LinkedIn where it processes tens of billions of messages per day.

Detailed documentation isn't available yet (we're working on it) but you can get a basic overview form this [Building LinkedIn’s Real-time Activity Data Pipeline](http://sites.computer.org/debull/A12june/pipeline.pdf "Building LinkedIn’s Real-time Activity Data Pipeline"). There is also a google group for discussion that you can email at camus_etl@googlegroups.com, <camus_etl@googlegroups.com> or you can search the [archives](https://groups.google.com/forum/#!forum/camus_etl "Camus Archives"). If you are interested please ask any questions on that mailing list.

# Brief Overview
All work is done within a single Hadoop job divided into three stages:

1. Setup stage fetches available topics and partitions from Zookeeper and the latest offsets from the Kafka Nodes.

2. Hadoop job stage allocates topic pulls among a set number of tasks.  Each task does the following:
*  Fetch events  from Kafka server and collect count statistics..
*  Move data  files to corresponding directories based on time stamps of events.
*  Produce count  events and write to HDFS.  * TODO: Determine status of open sourcing  Kafka Audit.
*  Store updated  offsets in HDFS.

3. Cleanup stage reads counts from all tasks, aggregates the values, and submits the results to Kafka for consumption by Kafka Audit. 

## Setup Stage 

1. Setup stage fetches from Zookeeper Kafka broker urls and topics (in /brokers/id, and /brokers/topics).  This data is transient and will be gone once Kafka server is down.

2. Topic offsets stored in HDFS.  Camus maintains its own status by storing offset for each topic in HDFS. This data is persistent.

3. Setup stage allocates all topics and partitions among a fixed number of tasks.

## Hadoop Stage 

### 1. Pulling the Data 

Each hadoop task uses a list of topic partitions with offsets generated by setup stage as input. It uses them to initialize Kafka requests and fetch events from Kafka brokers. Each task generates four types of outputs (by using a custom MultipleOutputFormat):
Avro data files;
Count statistics files;
Updated offset files.
Error files. * Note, each task generates an error file even if no errors were  encountered.  If no errors occurred, the file is empty.

### 2. Committing the data 

Once a task has successfully completed, all topics pulled are committed to their final output directories. If a task doesn't complete successfully, then none of the output is committed.  This allows the hadoop job to use speculative execution.  Speculative execution happens when a task appears to be running slowly.  In that case the job tracker then schedules the task on a different node and runs both the main task and the speculative task in parallel.  Once one of the tasks completes, the other task is killed.  This prevents a single overloaded hadoop node from slowing down the entire ETL.

### 3. Producing Audit Counts 

Successful tasks also write audit counts to HDFS. 

### 4. Storing the Offsets 

Final offsets are written to HDFS and consumed by the subsequent job.

## Job Cleanup 

Once the hadoop job has completed, the main client running on Azkaban reads all the written audit counts and aggregates them.  The aggregated results is then submitted to Kafka.

## Configurations 

Here is an abbreviated list of commonly used parameters.

Note, these are azkaban property files, soon to be replaced with something more generic.  You will notice all properties are prepended with “hadoop-conf.”  The Azkaban job adds these properties to the hadoop job configuration with the prepend remove.  i.e. hadoop-conf.zookeeper.hosts is added to the job conf as zookeeper.hosts.

* Zookeeper configurations:
 * hadoop-conf.zookeeper.hosts
 * hadoop-conf.zookeeper.broker.topics/brokers/topics
 * hadoop-conf.zookeeper.broker.nodes/brokers/ids
* blacklist/whitelist
 * hadoop-conf.kafka.blacklist.topics
 * hadoop-conf.kafka.whitelist.topics
* url to schema registry
 * hadoop-conf.etl.schema.registry.url
* time granularity of outputs
 * hadoop-conf.etl.output.file.time.partition.mins10 
* kafka config: client buffer size 1M
 * hadoop-conf.kafka.client.buffer.size20971520
* kafka config: client timeout 1 minute
 * hadoop-conf.kafka.client.so.timeout60000 
* pull restrictions
 * hadoop-conf.kafka.max.pull.hrs6
 * hadoop-conf.kafka.max.historical.days3
 * hadoop-conf.kafka.max.pull.minutes.per.partition10
 * hadoop-conf.kafka.monitor.time.granularity10 
* Topic Output Base Path
 * hadoop-conf.etl.destination.path
* The location of the Jobs are run from
 * hadoop-conf.etl.execution.base.path 
* The location of previous output directories.
 * hadoop-conf.etl.execution.history.path 
* Audit props
 * hadoop-conf.etl.counts.path 
 * hadoop-conf.kafka.monitor.tier
