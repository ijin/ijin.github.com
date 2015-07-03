---
layout: post
title: "Data PipelineによるDynamoDBのexport"
date: 2015-07-02 20:28
comments: true
categories: 
- aws
---

ちょっとハマったのでメモ。

DynamoDBにはRDSみたいなスナップショットによるバックアップ機能がなく、データを一括でexportするにはフルスキャンしかありません。
AWSではData Pipelineによるs3へのexportテンプレートがあって、それを使うとEMRクラスタが立ち上がりHive経由で大量の処理をして、s3へ書き出してくれます。

1000万件程度の小さな件数だとデフォルトのテンプレートがそのまま使えるけど、1億件近くになると失敗したりタイムアウトしたりするので、パラメータの調整が必要になってきます。

### 前提

* 約1億件
* 20GB
* Provisioned Throughput (reads): 1000
* Read Throughput Percent: 1.0
* 2時間以内のexport

### エラー
具体的には以下のようなエラーに遭遇しました。

```
Caused by: org.apache.hadoop.hive.ql.metadata.HiveException: Hive Runtime Error
 while processing row {"item":{"id_list":"{\"l\":[{\"n\":\"958\"},
{\"n\":\"704\"},{\"n\":\"847\"},{\"n\":\"232\"},{\"n\":\"72\"}]}",
"code":"{\"s\":\"adarea9\"}","user_area":"{\"s\":\"91657-adarea9\"}",
"user":"{\"s\":\"91657\"}","app_code":"{\"s\":\"xxx\"}","last_seen_at":
"{\"s\":\"2010-06-23 22:57:49 +0000\"}","target_id":"{\"n\":\"395\"}",
"count":"{\"n\":\"44\"}","promo_id":"{\"n\":\"125\"}"}} at 
org.apache.hadoop.hive.ql.exec.MapOperator.process(MapOperator.java:550) at 
org.apache.hadoop.hive.ql.exec.mr.ExecMapper.map(ExecMapper.java:177) 
... 8 more Caused by: org.apache.hadoop.hive.ql.metadata.HiveException: 
java.io.IOException: All datanodes 10.160.102.191:9200 are bad. Aborting... at 
org.apache.hadoop.hive.ql.exec.FileSinkOperator.processOp(FileSinkOperator.java:651) at 
org.apache.hadoop.hive.ql.exec.Operator.forward(Operator.java:793) at org.apach
```

```
2015-06-29 13:33:31,610 Stage-1 map = 100%, reduce = 0% MapReduce Total cumulative 
CPU time: 14 minutes 19 seconds 0 msec Ended Job = job_1435578726935_0002 with 
errors Error during job, obtaining debugging information... 
Examining task ID: task_1435578726935_0002_m_000008 (and more) from job job_1435578726935_0002 
Examining task ID: task_1435578726935_0002_m_000000 (and more) from job job_1435578726935_0002 
Examining task ID: task_1435578726935_0002_m_000000 (and more) from job job_1435578726935_0002 
Task with the most failures(4): 
----- Task ID: task_1435578726935_0002_m_000003 URL: http://ip-10-160-25-23.ap-northeast-1.compute.
internal:9026/taskdetails.jsp?jobid=job_1435578726935_0002&tipid=task_1435578726935_0002_m_000003
----- Diagnostic Messages for this Task: Error: GC overhead limit exceeded FAILED: 
Execution Error, return code 2 from org.apache.hadoop.hive.ql.exec.mr.MapRedTask 
MapReduce Jobs Launched: Job 0: Map: 10 Cumulative CPU: 859.0 sec HDFS Read: 0 HDFS Write: 0 FAIL Total MapRed
```

```
2015-06-30 02:32:17,929 Stage-1 map = 100%, reduce = 0%, Cumulative CPU 1605.94 sec 
MapReduce Total cumulative CPU time: 26 minutes 45 seconds 940 msec Ended Job = 
job_1435627213542_0001 with errors Error during job, obtaining debugging information... 
Examining task ID: task_1435627213542_0001_m_000004 (and more) from job job_1435627213542_0001 
Examining task ID: task_1435627213542_0001_m_000004 (and more) from job job_1435627213542_0001 
Examining task ID: task_1435627213542_0001_m_000004 (and more) from job job_1435627213542_0001 
Task with the most failures(4): ----- Task ID: task_1435627213542_0001_m_000000 URL: 
http://ip-10-150-205-59.ap-northeast-1.compute.internal:9026/taskdetails.jsp?
jobid=job_1435627213542_0001&tipid=task_1435627213542_0001_m_000000 ----- 
Diagnostic Messages for this Task: AttemptID:attempt_1435627213542_0001_m_000000_3 
Timed out after 600 secs FAILED: Execution Error, return code 2 from 
org.apache.hadoop.hive.ql.exec.mr.MapRedTask MapReduce Jobs Launched: Job 0: Map: 10 Cu
```

```
Error: java.lang.RuntimeException: Hive Runtime Error while closing operators at 
org.apache.hadoop.hive.ql.exec.mr.ExecMapper.close(ExecMapper.java:260) at 
org.apache.hadoop.mapred.MapRunner.run(MapRunner.java:81) at 
org.apache.hadoop.mapred.MapTask.runOldMapper(MapTask.java:432) at 
org.apache.hadoop.mapred.MapTask.run(MapTask.java:343) at 
org.apache.hadoop.mapred.YarnChild$2.run(YarnChild.java:175) at 
java.security.AccessController.doPrivileged(Native Method) at 
javax.security.auth.Subject.doAs(Subject.java:415) at 
org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1548) at 
org.apache.hadoop.mapred.YarnChild.main(YarnChild.java:170) 
Caused by: org.apache.hadoop.hive.ql.metadata.HiveException: 
java.io.IOException: All datanodes 10.186.28.181:9200 are bad. 
Aborting... at org.apache.hadoop.hive.ql.exec.FileSinkOperator$FSPaths.closeWriters(FileSinkOperator.java:168) at 
org.apache.hadoop.hive.ql.exec.FileSinkOperator.closeOp(FileSinkOperator.java:882) at org.apache.hadoop.
```

Hive Runtime ErrorやGCエラー等が出てますね。

### 原因

通常のexportテンプレートではEMRはなんと**m1.medium**のcore taskが一台のみ起動して処理が走ります。
各種ヒープサイズの設定（`YARN_RESOURCEMANAGER_HEAPSIZE` 等）は[instance typeによって決まって](https://docs.aws.amazon.com/ElasticMapReduce/latest/DeveloperGuide/HadoopMemoryDefault_H2.html)おり、この件数とデータサイズではHEAPSIZEが不足し、GCエラー等が発生してOut of Memory状態になって処理が落ちるようです。
そこで、メモリをもっと搭載している大きめのインスタンスでHEAPSIZEを確保してあげる必要があります。

### 解決策

こんな感じ。

{% img https://lh3.googleusercontent.com/ghExdELQB1cIE9K-d97gDqz7a634413GOTxAHhvJsBg %}

|parameter|default value|new value|
|-|-|
|core & master instance type|m1.medium|m3.xlarge
|core instance count|1|2
|AMI version|3.3.2|3.8.0

m3.xlargeにした事で処理が落ちる事なくスムーズに実行されるようになりました。core countを増やしたのは、exportテンプレートのデフォルト設定だとcountが1なので、mapperが不足し、DynamoDBで設定したthroughput (1000)をフルに使い切る事ができなく、デフォルトのタイムアウト時間（2時間）に達して処理自体がキャンセルされてしまうからです。EMR側のスループットも上げる為に必要な変更でした。また、Hadoopの[AMI version](http://docs.aws.amazon.com/ElasticMapReduce/latest/DeveloperGuide/ami-versions-supported.html)も古い3.3.2から最新の3.8.0にしてあります。

これらによって、export処理がだいたい1時間ちょいで完了します。

その時の試行錯誤がこれ。

{%img https://lh3.googleusercontent.com/xatqR0hx1M6PhroKFNAksOplqgqYgqsaCJgonanQyzA=w437-h197-no %}

### 計算

provisioned throughputに対するバックアップ時間の目安を計算するには以下の通り。

20GBのデータ、1億件のレコード、1000 throughputとして、

`平均item size = 20*1024*1024*1024/100000000 =~  215 byte`

4KB以下なので4KB blockのreadとなります。
Hive Queryのread処理は[eventually consistentになる](http://hipsterdevblog.com/blog/2015/05/30/analysing-dynamodb-index-usage-in-hive-queries/)ので、1 IOPSに対して8KBのデータが読み込めます。
そうすると

`DynamoDB scan時間 = (20*1024*1024*1024)/(1000*8*1024*60) = 43.7分`

実際にはEMRクラスタのオーバーヘッドが20分弱程度あるので、これに若干加算します。

図にするとこんな風になります。

<iframe width="670" height="412" seamless frameborder="0" scrolling="no" src="https://docs.google.com/spreadsheets/d/14oyOry-bnrqIgYxdN36Qdwv3VC1lhdku4QffzFh8iIc/pubchart?oid=1201814511&amp;format=interactive"></iframe>

バックアップのタイミングでDynamoDBのthroughputをガツンと上げれば、件数によっては短時間で済む場合もあるので、参考にでも！

（※ Production Trafficに影響がないように、Read Throughput Percentは適切に設定する必要があります）

