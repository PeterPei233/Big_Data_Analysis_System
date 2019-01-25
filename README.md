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

### 2.2 Build googleplay crwaler
write our own parser and add it in Parse-plugin.xml 
```
# go to CS502-1801
git clone https://github.com/apache/nutch
cd nutch
git checkout release-1.12
# apply patch
patch -p1<.../googleplaycrawler.patch
ant job
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


# 2.Web Crawler with Native Hadoop (Nutch)
