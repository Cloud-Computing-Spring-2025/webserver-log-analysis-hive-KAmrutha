## Web Server Log Analysis using Hive and HDFS ##

## Project Overview

This project focuses on analyzing web server logs using Apache Hive and Hadoop Distributed File System (HDFS). The primary objectives include:
1. Loading and querying log data efficiently.
2. Partitioning data for optimized query performance.
3. Extracting insights such as total requests, most visited pages, status code distribution, traffic trends, and identifying suspicious IPs.
4. Storing analysis results in HDFS for further use.

## Implementation Approach
The implementation follows these key steps:

1. Data Loading into HDFS

The web server log file web_server_logs.csv is copied to the Hadoop environment and uploaded into HDFS.
``
docker cp web_server_logs.csv resourcemanager:/opt/hadoop-2.7.4/share/hadoop/mapreduce/
docker exec -it resourcemanager /bin/bash
cd /opt/hadoop-2.7.4/share/hadoop/mapreduce/
hdfs dfs -mkdir /input
hdfs dfs -put web_server_logs.csv /input
hdfs dfs -ls /input/*
``
2. Creating External Table in Hive
An external Hive table web_logs_external is created to reference the raw log data stored in HDFS.
``
CREATE EXTERNAL TABLE IF NOT EXISTS web_logs_external (
    ip STRING,
    log_timestamp STRING,
    url STRING,
    status INT,
    user_agent STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
STORED AS TEXTFILE LOCATION '/input/';
``
Query to check data
``
SELECT * FROM web_logs_external LIMIT 10;
``
3. Creating Partitioned Table

A partitioned table partitioned_web_logs is created with status as a partition column.
``
CREATE TABLE IF NOT EXISTS partitioned_web_logs (
    ip STRING,
    log_timestamp STRING,
    url STRING,
    user_agent STRING
)
PARTITIONED BY (status INT)
STORED AS TEXTFILE;

SET hive.exec.dynamic.partition=true;
SET hive.exec.dynamic.partition.mode=nonstrict;
``
4. Loading Data into Partitioned Table
``
INSERT OVERWRITE TABLE partitioned_web_logs
PARTITION(status)
SELECT ip, log_timestamp, url, user_agent, status
FROM web_logs_external;
``
## Querying Data for Analysis

1. Total Requests Count
``
SELECT COUNT(*) AS total_requests FROM partitioned_web_logs;
``

2. Status Code Distribution
``
SELECT
    status,
    COUNT(*) AS request_count
FROM partitioned_web_logs
GROUP BY status
ORDER BY request_count DESC;
``
3. Top 3 Most Visited Pages
``
SELECT
    url,
    COUNT(*) AS visit_count
FROM partitioned_web_logs
GROUP BY url
ORDER BY visit_count DESC
LIMIT 3;
``
4. Traffic Sources by User-Agent
``
SELECT
    user_agent,
    COUNT(*) AS agent_count
FROM partitioned_web_logs
GROUP BY user_agent
ORDER BY agent_count DESC;
``
5. Identifying Suspicious IPs with High Failure Requests
``
SELECT
    ip,
    COUNT(*) AS failed_request_count
FROM partitioned_web_logs
WHERE status = 404 OR status = 500
GROUP BY ip
HAVING COUNT(*) > 3
ORDER BY failed_request_count DESC;
``
6. Traffic Trends Over Time
``
SELECT CONCAT_WS(' ', time_minute, CAST(request_count AS STRING))
FROM (
    SELECT SUBSTR(log_timestamp, 0, 16) AS time_minute, COUNT(*) AS request_count
    FROM partitioned_web_logs
    GROUP BY SUBSTR(log_timestamp, 0, 16)
    ORDER BY time_minute
) AS subquery;
``

### Storing Analysis Results in HDFS

The analysis results are stored in designated HDFS directories:
``
INSERT OVERWRITE DIRECTORY '/output/web_logs_analysis/web_requests'
SELECT COUNT(*) AS total_requests FROM partitioned_web_logs;

INSERT OVERWRITE DIRECTORY '/output/web_logs_analysis/status_codes'
SELECT
    status,
    COUNT(*) AS request_count
FROM partitioned_web_logs
GROUP BY status
ORDER BY request_count DESC;

INSERT OVERWRITE DIRECTORY '/output/web_logs_analysis/visited_pages'
SELECT
    url,
    COUNT(*) AS visit_count
FROM partitioned_web_logs
GROUP BY url
ORDER BY visit_count DESC
LIMIT 3;

INSERT OVERWRITE DIRECTORY '/output/web_logs_analysis/traffic_sources'
SELECT
    user_agent,
    COUNT(*) AS agent_count
FROM partitioned_web_logs
GROUP BY user_agent
ORDER BY agent_count DESC;

INSERT OVERWRITE DIRECTORY '/output/web_logs_analysis/suspicious_ips'
SELECT
    ip,
    COUNT(*) AS failed_request_count
FROM partitioned_web_logs
WHERE status = 404 OR status = 500
GROUP BY ip
HAVING COUNT(*) > 3
ORDER BY failed_request_count DESC;

INSERT OVERWRITE DIRECTORY '/output/web_logs_analysis/traffic_trends'
SELECT CONCAT_WS(' ', time_minute, CAST(request_count AS STRING))
FROM (
    SELECT SUBSTR(log_timestamp, 0, 16) AS time_minute, COUNT(*) AS request_count
    FROM partitioned_web_logs
    GROUP BY SUBSTR(log_timestamp, 0, 16)
    ORDER BY time_minute
) AS subquery;
``

### Export query results 

``
docker exec -it resourcemanager /bin/bash
cd /opt/hadoop-2.7.4/share/hadoop/mapreduce/
hdfs dfs -get /output /opt/hadoop-2.7.4/share/hadoop/mapreduce/
exit
docker cp resourcemanager:/opt/hadoop-2.7.4/share/hadoop/mapreduce/output/ ./output/

``

## Challenges Faced

1. Data Format Issues: Some records had missing values or incorrect delimiters. Resolved by preprocessing data before ingestion.
2. Partitioning Errors: Initially, partitioning did not work due to dynamic partition settings. Fixed by enabling dynamic partitions.
3. Performance Optimization: Query execution was slow on large data. Optimized by partitioning data on status and using LIMIT for sampling.
4. HDFS File Visibility: After inserting results into HDFS, files did not appear immediately. Resolved by checking permissions and using hdfs dfs -ls to verify paths.

## Sample Input & Output
Sample Input  - web_server_logs.csv

## Output
1. 
``
 	total_requests
1	101
``
2. 
`` 
 	status	request_count
1	200	57
2	500	22
3	404	21
4	NULL	1
``

3. 
``
  	url	visit_count
1	/products	24
2	/contact	24
3	/home	20
``
4. 
``
 	user_agent	agent_count
1	Edge/88.0	23
2	Chrome/90.0	23
3	Opera/74.0	21
4	Safari/13.1	17
5	Mozilla/5.0	16
6	user_agent	1

``
5. 
``
 	ip	failed_request_count
1	192.168.1.3	9
2	192.168.1.15	7
3	192.168.1.20	6
4	192.168.1.2	6
5	192.168.1.1	5
6	192.168.1.25	4
``
6. 
``
 	_c0
1	2024-02-01 10:15 7
2	2024-02-01 10:16 2
3	2024-02-01 10:17 3
``