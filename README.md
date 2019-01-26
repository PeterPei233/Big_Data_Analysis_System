# 1.Overview


# 2.Web Crawler with Native Hadoop (Nutch)

crawler app info from google Play

### 2.1 Structure of Nutch

- Inject: inject seed to nutchdb
- Generate: generate urls to crawl from nutchdb
- Fetch: Crawl html pages
- Parse: Extract metadata and outlinks
- Update: Update nutchdb with new outlinks
![img](https://github.com/PeterPei666/Real-time_Log_Analysis_System/blob/master/img/nutch.png)

### 2.2 Build googleplay crawler

replace the default crawler with googlepalycrawler

```
patch -p1<.../googleplaycrawler.patch
```

Fix Skew 
```
patch -p1</xxxx/googleplaycrawler/fixskew.patch
ant job
```
Run googleplaycrawler locally
```
hadoop jar build/apache-nutch-1.12.job org.apache.nutch.googleplay.GooglePlayCrawler -
Dmapreduce.framework.name=local -Dfs.defaultFS=file:/// seed -numFetchers 100 -depth 2
```

Run googleplaycrawler on a Single Node Cluster

```
hadoop jar build/apache-nutch-1.12.job
org.apache.nutch.googleplay.GooglePlayCrawler seed -numFetchers 10
# check job status in WebUI, notice fetch job has skew
# check output
export HADOOP_CLASSPATH=/src/nutch/build/apache-nutch-1.12.job
hadoop fs -text nutchdb/segments/xxxxxxxx/parse_data/part-00000/data
```

Run Google Play in AWS

Upload jar and seed to S3 --> Start an EMR job --> Add Tasks on Demand


# 3 DataProcessing Samples (Background Knowledge)

### 3.1 Pig

run pig scrpit

```
--first.pig
a= load 'studenttab10k' as (name:chararray, age:int, gpa:double);
b = filter a by age > 18;
c = foreach b generate name, ROUND(gpa) as gpa;
d = group c by name;
e = foreach d generate group, AVG(c.gpa) as gpa;
f = distinct e;
dump f;
```

```
pig first.pig
```

```
# run with Tez
pig -x tez
```

### 3.2 Spark

Start Spark Standalone Cluster
```
cd ~/spark-2.3.0
sbin/start-master.sh
# replace spark://xxxxxxxxxxxx:7077 with your own
sbin/start-slave.sh spark://xxxxxxxxxxxx:7077
```

run Spark DataFrame Example
```
spark-submit --master spark://xxxxxxxxxxxx:7077 --class DataFrameTest
target/scala-example-1.0-SNAPSHOT.jar
```
Spark on Yarn
```
# Stop standalone Spark cluster
sbin/stop-slave.sh
sbin/stop-master.sh
# Start Spark history server
sbin/start-history-server.sh
spark-submit --master yarn --class DataFrameTest target/scala-example-1.0-SNAPSHOT.jar
```

### 3.3 Hive/Presto Samples - SQL Engine

Hive - large scale

Presto - MPP on Hadoop

### 3.4 Flink - Stream processing

Similar to Spark, but more focused on streaming data processing

# 4. Big Data analysis System
**Data Source:**  Real Access log (180G): https://s3.amazonaws.com/cloudacl/access_log/index.html

### 4.1 Data Processing in Pig
Target:  Find the most frequently visited apps category in US

1. Filter: Android App Only 
2. Extract Fields with regular Expression   (register UDF GeoIP: mapping IP --> country, city)
3. Write EvalFunc to call GeoIP library and get the location
4.Ship Files to Distributed Cache

```
public List<String> getShipFiles() {
    List<String> shipFiles = new ArrayList<String>();
    shipFiles.add("GeoLiteCity.dat");
    return shipFiles;
}
```
5. Duplicate and filter(leave log only in usa)
6. Load NutchDb (the previous crwaled data from GooglePlay) from s3 
```
b1 = load ‘s3a://daijytest/nutchdb/segments/*/parse_data/part- */data' using com.example.pig.NutchParsedDataLoader()
```
7. Extract App Id and categrory from crawled data and join with log data
```
b3 = foreach b2 generate flatten(REGEX_EXTRACT_ALL(url, '.*?id=(.*)?')) as (id:chararray), category;
c = join a8 by id, b3 by id;
```
8.  Aggregation on Category and sort

```
d = group c by category;
e = foreach d generate group, COUNT(c) as count;
f = order e by count desc;
```

9. Part Result (only with small part of access log)
```
(Communication,410)
(Tools,332)
(Productivity,213)
(Social,180)
(Entertainment,157)
(Education,101)
(Music and Audio,100)
(Action,88)
(Books and Reference,71)
(Finance,70
```

10. Run Pig on AWS

### 4.2 Data Processing in Spark

Similar pipeline as working in pig

Different from Pig --> Don't need to write DataLoader in person to load sequence file
```
JavaPairRDD<Text, ParseData> inputRDD = context.sequenceFile("s3a://daijytest/nutchdb/segments/*/parse_data/part-*/data", Text.class, ParseData.class);
```
```
spark-submit --class com.example.spark.GetTopCategory --jars ../resource/nutch-1.12.jar,/home/hadoop/hadoop- 2.9.0/share/hadoop/tools/lib/hadoop-aws- 2.9.0.jar,/home/hadoop/hadoop-2.9.0/share/hadoop/tools/lib/aws- java-sdk-bundle-1.11.199.jar,../resource/geoip-api-1.3.1.jar -- files ../resource/GeoLiteCity.dat target/gettopcategory-1.0- SNAPSHOT.jar
```
- run Spark on AWS

Configure
```
Deploy mode: Client
Spark-submit options: --class com.example.spark.GetTopCategory --jars s3://cloudacl/code/nutch-1.12.jar,s3://cloudacl/code/geoip-api-1.3.1.jar --files s3://cloudacl/code/GeoLiteCity.dat
Application location: s3://cloudacl/code/gettopcategory-1.0-SNAPSHOT.jar
```

### 4.3 Real-time Analysis System 

Input data: access log

Target : Find the top 10 countries which have the most total visits

![img](https://github.com/PeterPei666/Real-time_Log_Analysis_System/blob/master/img/real_time.png)

Target: find
Simulate log glowing:

**1. Real-time Analysis by Fink** 

a.simulate log coming in

```
/src/week3/lab2/flink/log/process.py /src/week3/lab2/flink/log/sample.txt >
localhost_access_log.xxxx.txt
```

b. Data Ingestion
```
dataStream = env.readFileStream(path, Time.seconds(1).toMilliseconds(), WatchType.PROCESS_ONLY_APPENDED)
```

c. Flink Data Flow

```
dataStream.map(country, dt, cat)
          .keyBy(country, cat)
          .window(TumblingEventTimeWindows.of(Time.hours(1))).count;
```

d. Extract info and save in HDFS

**2. Batch Data Analysis by Hive** 

a. Creat Partitioned Table:

```
create table access_log_partitioned(country string, dt timestamp, cat string, count int) partitioned by(d date) row format serde 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe' with serdeproperties('field.delim'='\t', 'timestamp.formats'='yyyy-MM-dd HH:mm:ss') stored as textfile;
```

b. Extract Info

c. Results
```
United States 900079276
Peru 85283216
Philippines 50572854
Mexico 23807685
Colombia 21391952
Italy 18015359
Bolivia 17605752
Venezuela 14599674
Ecuador 12835092
Argentina 12022976
```
