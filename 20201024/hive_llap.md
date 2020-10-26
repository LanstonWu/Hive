Hive LLAP
===

***内容目录***  
1 hive llap该怎么部署  
2 注意事项  
3 llap初始化  
4 性能测试  
5 总结  
链接  


> 从Hive llap特性的出现，分析作用、部署、使用细节问题，总结hive llap使用经验和注意事项。
(From the appearance of the Hive llap feature, analyze the function, deployment, and use details, and summarize the experience and precautions of the hive llap)

  LLAP是hive 2.0.0版本引入的新特性，hive官方称为(Live long and process)，hortonworks公司的CDH称为(low-latency analytical processing)，其实它们都是一样的，都是实现将数据预取、缓存到基于yarn运行的守护进程中，降低和减少系统IO和与HDFS DataNode的交互，具体的特性细节参考官方文档 [Hive llap](https://cwiki.apache.org/confluence/display/Hive/LLAP) (如果链接未生效，在文章后面的链接中获取)，但是***由于版本更新频繁和官方文档的维护不力因素，很多地方和使用上让人概念不清、正确和错误分不清***，特别是用CDH这样的集成套件，很多细节被忽略，本文一一来细说和总结各类问题。

## 1 hive llap该怎么部署
分两种情况：
1. 如果使用的hadoop yarn版本是3.1.0以下(不包含3.1.0)，需要使用 [Apache slider](http://incubator.apache.org/projects/slider.html) 来部署，因为在hadoop yarn 3.1.0之前，yarn本身不支持长时间运行的服务(long running services)，而slider组件是可以打包、管理和部署长时间运行的服务到yarn上运行的。
2. 如果使用的hadoop yarn版本是3.1.0及以上，完全不需要slider组件了，因为从 [hadoop yarn 3.1.0](http://hadoop.apache.org/docs/r3.1.0) 开始，yarn已经合并支持long running services了，slider项目也停止更新了。

因此，部署时要考虑使用的组件版本，再确定部署方案，对于开源项目，使用的版本和环境很重要，如果组件本身已经提供特性和功能，并且一直处于维护状态，建议尽量不要使用其它组件替代，替代成本和异常问题远比想象的高。    
当然，如果你使用的是CDH类的集成套件，套件本身已经集成封装，每个套件版本会提供相应的支持，这些内容就无需多虑了。

## 2 注意事项
1. llap 目前只支持tez引擎，需要先部署好hive和tez；
2. 由于llap依赖zookeeper和hadoop组件，如果集群开启了安全认证(比如kerberos)，llap也要进行安全认证相关配置，使用到的配置参数如：

```xml
<property>
	<name>hive.llap.daemon.keytab.file</name>
	<value>/etc/security/keytabs/demo.keytab</value>
</property>

<property>
	<name>hive.llap.daemon.service.principal</name>
	<value>demo/sywu@sywukeb</value>
</property>

<property>
	<name>hive.llap.task.scheduler.am.registry.keytab.file</name>
	<value>/etc/security/keytabs/demo.service.keytab</value>
</property>

<property>
	<name>hive.llap.task.scheduler.am.registry.principal</name>
	<value>demo/sywu@sywukeb</value>
</property>
```
另外随着程序的更新，官方文档上的参数参差不齐，***有些参数需要阅读和从代码中查找***。

3. 一些网上资料和CDH文档的部署方式使用hive用户和权限运行llap服务，hive的权限很大，如果集群很大，使用的人很多，对权限控制粒度要求高，不适合使用这种方式，应该考虑多个llap服务，为不同的用户或者项目组开放不同的llap服务。
4. 由于LLAP所具有的优势(预取、缓存)，对于大的集群，考虑面向不同场景和用户使用不同的llap服务，提高查询命中率，提升性能，我认为是合理的。

## 3 llap初始化
以下以hadoop 3.1.0，hive 3.1.0，tez 0.9.1，集群无安全认证为例，首先配置llap；
```xml
<property>
	<name>hive.llap.execution.mode</name>
	<value>all</value>
</property>
<property>
	<name>hive.execution.mode</name>
	<value>llap</value>
</property>

<property>
	<name>hive.llap.daemon.service.hosts</name>
	<value>@sywu-llap01</value>
</property>

<property>
	<name>hive.llap.daemon.memory.per.instance.mb</name>
	<value>25600</value>
</property>

<property>
	<name>hive.llap.daemon.num.executors</name>
	<value>8</value>
</property>

<property>
	<name>hive.llap.zk.sm.connectionString</name>
	<value>sywu01:2181</value>
</property>

<property>
	<name>hive.llap.zk.registry.namespace</name>
	<value>hive_sywu01</value>
</property>

<property>
	<name>hive.llap.zk.registry.user</name>
	<value>sywu</value>
</property>
```
hive.llap.daemon.service.hosts 配置llap 实例名称，这个名称和启动的名称相同。然后打包和准备部署llap 服务的文件；
```xml
hive --service llap --name sywu-llap01 --instances 4 --size 60g --loglevel info --cache 30g --executors 10 --iothreads 10 --args " -XX:+UseG1GC -XX:+ResizeTLAB -XX:+UseNUMA -XX:-ResizePLAB" 
```
这个命令会在当前目录生成llap 服务文件夹，里面包含启动llap的脚本，llap的相关配置和jar包；
```xml
$ ll
total 184M
-rwxr-xr-x 1 sywu01 sywu01 184M Oct 20 15:28 llap-20Oct2020.tar.gz
-rwxr-xr-x 1 sywu01 sywu01  273 Oct 20 15:28 run.sh
-rwxr-xr-x 1 sywu01 sywu01 2.0K Oct 20 15:29 Yarnfile
```
执行run.sh 文件启动llap服务。到此llap部署到yarn上并运行。
```xml
LLAPSTATUS
--------------------------------------------------------------------------------
LLAP Application running with ApplicationId=application_1602234006497_0592
--------------------------------------------------------------------------------
LLAP Application running with ApplicationId=application_1602234006497_0592
--------------------------------------------------------------------------------

{
  "amInfo" : {
    "appName" : "sywu-llap01",
    "appType" : "yarn-service",
    "appId" : "application_1602234006497_0592"
  },
  "state" : "RUNNING_ALL",
  "desiredInstances" : 4,
  "liveInstances" : 4,
  "launchingInstances" : 0,
  "appStartTime" : 0,
  "runningThresholdAchieved" : false,
  "runningInstances" : [ {
    "hostname" : "sywu01",
    "containerId" : "container_e48_1602234006497_0592_01_000013",
    "statusUrl" : "http://sywu01:15002/status",
    "webUrl" : "http://sywu01:15002",
    "rpcPort" : 45795,
    "mgmtPort" : 15004,
    "shufflePort" : 15551,
    "yarnContainerExitStatus" : 0
  }, {
    "hostname" : "sywu02",
    "containerId" : "container_e48_1602234006497_0592_01_000005",
    "statusUrl" : "http://sywu02:15002/status",
    "webUrl" : "http://sywu02:15002",
    "rpcPort" : 46845,
    "mgmtPort" : 15004,
    "shufflePort" : 15551,
    "yarnContainerExitStatus" : 0
  }, {
    "hostname" : "sywu01",
    "containerId" : "container_e48_1602234006497_0592_01_000008",
    "statusUrl" : "http://sywu01:15002/status",
    "webUrl" : "http://sywu01:15002",
    "rpcPort" : 33382,
    "mgmtPort" : 15004,
    "shufflePort" : 15551,
    "yarnContainerExitStatus" : 0
  }, {
    "hostname" : "sywu03",
    "containerId" : "container_e48_1602234006497_0592_01_000010",
    "statusUrl" : "http://sywu03:15002/status",
    "webUrl" : "http://sywu03:15002",
    "rpcPort" : 43520,
    "mgmtPort" : 15004,
    "shufflePort" : 15551,
    "yarnContainerExitStatus" : 0
  } ]
}
```

## 4 性能测试
到此，hive已经有mr和tez引擎，并支持llap，使用hortonworks公司开源的 [hive-testbench项目](https://github.com/hortonworks/hive-testbench) 生成1Tb数据；
```xml
 $ ./tpcds-setup.sh 1000
```
用query10.sql 中的关联脚本查询测试；
```sql
select  
  cd_gender,cd_marital_status,cd_education_status,count(*) cnt1,cd_purchase_estimate,count(*) cnt2,cd_credit_rating,count(*) cnt3,cd_dep_count,count(*) cnt4,cd_dep_employed_count,count(*) cnt5,cd_dep_college_count,count(*) cnt6
 from
  customer c,customer_address ca,customer_demographics
 where
  c.c_current_addr_sk = ca.ca_address_sk and
  ca_county in ('Fillmore County','McPherson County','Bonneville County','Boone County','Brown County') and
  cd_demo_sk = c.c_current_cdemo_sk and 
  exists (select *
          from store_sales,date_dim
          where c.c_customer_sk = ss_customer_sk and
                ss_sold_date_sk = d_date_sk and
                d_year = 2000 and
                d_moy between 3 and 3+3) and
   (exists (select *
            from web_sales,date_dim
            where c.c_customer_sk = ws_bill_customer_sk and
                  ws_sold_date_sk = d_date_sk and
                  d_year = 2000 and
                  d_moy between 3 ANd 3+3) or 
    exists (select * 
            from catalog_sales,date_dim
            where c.c_customer_sk = cs_ship_customer_sk and
                  cs_sold_date_sk = d_date_sk and
                  d_year = 2000 and
                  d_moy between 3 and 3+3))
 group by cd_gender,
          cd_marital_status,
          cd_education_status,
          cd_purchase_estimate,
          cd_credit_rating,
          cd_dep_count,
          cd_dep_employed_count,
          cd_dep_college_count
 order by cd_gender,
          cd_marital_status,
          cd_education_status,
          cd_purchase_estimate,
          cd_credit_rating,
          cd_dep_count,
          cd_dep_employed_count,
          cd_dep_college_count
limit 100;
```
mr 引擎执行情况；
```xml
INFO  : Query ID = sywu01_20201023113411_add04434-7382-4376-8883-26ab298b1c6f
INFO  : Total jobs = 8
INFO  : Starting task [Stage-24:MAPREDLOCAL] in parallel
INFO  : Starting task [Stage-25:MAPREDLOCAL] in parallel
INFO  : Starting task [Stage-26:MAPREDLOCAL] in parallel
INFO  : Starting task [Stage-27:MAPREDLOCAL] in parallel
INFO  : Launching Job 1 out of 8
INFO  : Starting task [Stage-20:MAPRED] in parallel
INFO  : Launching Job 2 out of 8
INFO  : Starting task [Stage-14:MAPRED] in parallel
INFO  : Launching Job 3 out of 8
INFO  : Starting task [Stage-11:MAPRED] in parallel
INFO  : Launching Job 4 out of 8
INFO  : Starting task [Stage-18:MAPRED] in parallel
INFO  : Starting task [Stage-17:CONDITIONAL] in parallel
INFO  : Launching Job 5 out of 8
INFO  : Starting task [Stage-3:MAPRED] in parallel
INFO  : Launching Job 6 out of 8
INFO  : Starting task [Stage-4:MAPRED] in parallel
INFO  : Launching Job 7 out of 8
INFO  : Starting task [Stage-5:MAPRED] in parallel
INFO  : MapReduce Jobs Launched: 
INFO  : Stage-Stage-18: Map: 3   Cumulative CPU: 779.28 sec   HDFS Read: 77140427 HDFS Write: 5298244 SUCCESS
INFO  : Stage-Stage-20: Map: 350   Cumulative CPU: 4134.07 sec   HDFS Read: 3203667384 HDFS Write: 193140638 SUCCESS
INFO  : Stage-Stage-11: Map: 153  Reduce: 151   Cumulative CPU: 4631.57 sec   HDFS Read: 886558268 HDFS Write: 46326191 SUCCESS
INFO  : Stage-Stage-14: Map: 257  Reduce: 271   Cumulative CPU: 6646.95 sec   HDFS Read: 1049371661 HDFS Write: 106287345 SUCCESS
INFO  : Stage-Stage-3: Map: 19  Reduce: 2   Cumulative CPU: 394.45 sec   HDFS Read: 351370942 HDFS Write: 1399528 SUCCESS
INFO  : Stage-Stage-4: Map: 2  Reduce: 1   Cumulative CPU: 15.71 sec   HDFS Read: 1415039 HDFS Write: 12296 SUCCESS
INFO  : Stage-Stage-5: Map: 1  Reduce: 1   Cumulative CPU: 8.31 sec   HDFS Read: 23606 HDFS Write: 7168 SUCCESS
INFO  : Total MapReduce CPU Time Spent: 0 days 4 hours 36 minutes 50 seconds 340 msec
INFO  : Completed executing command(queryId=sywu01_20201023113411_add04434-7382-4376-8883-26ab298b1c6f); Time taken: 838.106 seconds
INFO  : OK
+------------+--------------------+-----------------------+-------+-----------------------+-------+-------------------+-------+---------------+-------+------------------------+-------+-----------------------+-------+
| cd_gender  | cd_marital_status  |  cd_education_status  | cnt1  | cd_purchase_estimate  | cnt2  | cd_credit_rating  | cnt3  | cd_dep_count  | cnt4  | cd_dep_employed_count  | cnt5  | cd_dep_college_count  | cnt6  |
+------------+--------------------+-----------------------+-------+-----------------------+-------+-------------------+-------+---------------+-------+------------------------+-------+-----------------------+-------+
| F          | D                  | 2 yr Degree           | 1     | 500                   | 1     | Good              | 1     | 0             | 1     | 0                      | 1     | 4                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 500                   | 1     | Good              | 1     | 1             | 1     | 0                      | 1     | 5                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 500                   | 1     | Good              | 1     | 1             | 1     | 1                      | 1     | 1                     | 1     |
....
| F          | D                  | 2 yr Degree           | 1     | 3500                  | 1     | High Risk         | 1     | 0             | 1     | 4                      | 1     | 4                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 3500                  | 1     | High Risk         | 1     | 3             | 1     | 2                      | 1     | 1                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 3500                  | 1     | High Risk         | 1     | 5             | 1     | 4                      | 1     | 0                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 3500                  | 1     | High Risk         | 1     | 5             | 1     | 6                      | 1     | 6                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 3500                  | 1     | Low Risk          | 1     | 0             | 1     | 3                      | 1     | 4                     | 1     |
+------------+--------------------+-----------------------+-------+-----------------------+-------+-------------------+-------+---------------+-------+------------------------+-------+-----------------------+-------+
100 rows selected (850.686 seconds)
```
tez 引擎执行情况；
```xml
INFO  : Query ID = sywu01_20201023175253_265d5780-7be4-47ad-ad4b-ef8154bb3842
INFO  : Total jobs = 1
INFO  : Launching Job 1 out of 1
INFO  : Starting task [Stage-1:MAPRED] in parallel
----------------------------------------------------------------------------------------------
        VERTICES      MODE        STATUS  TOTAL  COMPLETED  RUNNING  PENDING  FAILED  KILLED  
----------------------------------------------------------------------------------------------
Map 6 .......... container     SUCCEEDED     14         14        0        0       0      10  
Map 7 .......... container     SUCCEEDED      4          4        0        0       3       3  
Map 1 .......... container     SUCCEEDED      6          6        0        0       0       1  
Map 9 .......... container     SUCCEEDED      1          1        0        0       0       1  
Reducer 5 ...... container     SUCCEEDED      1          1        0        0       0       1  
Map 8 .......... container     SUCCEEDED     20         20        0        0       0       0  
Map 10 ......... container     SUCCEEDED      6          6        0        0       0       0  
Reducer 11 ..... container     SUCCEEDED    234        234        0        0       0       0  
Map 12 ......... container     SUCCEEDED     10         10        0        0       0       0  
Reducer 13 ..... container     SUCCEEDED    234        234        0        0       0       0  
Reducer 2 ...... container     SUCCEEDED    234        234        0        0       0       0  
Reducer 3 ...... container     SUCCEEDED    145        145        0        0       0       0  
Reducer 4 ...... container     SUCCEEDED      1          1        0        0       0       0  
----------------------------------------------------------------------------------------------
VERTICES: 13/13  [==========================>>] 100%  ELAPSED TIME: 36.01 s    
----------------------------------------------------------------------------------------------
INFO  : Completed executing command(queryId=sywu01_20201023175253_265d5780-7be4-47ad-ad4b-ef8154bb3842); Time taken: 54.039 seconds
INFO  : OK

- Query Execution Summary
- ----------------------------------------------------------------------------------------------
- OPERATION                            DURATION
- ----------------------------------------------------------------------------------------------
- Compile Query                           0.00s
- Prepare Plan                            0.00s
- Get Query Coordinator (AM)              0.00s
- Submit Plan                         1603446798.35s
- Start DAG                               1.05s
- Run DAG                                34.93s
- ----------------------------------------------------------------------------------------------
- 
- Task Execution Summary
- ----------------------------------------------------------------------------------------------
-   VERTICES      DURATION(ms)   CPU_TIME(ms)    GC_TIME(ms)   INPUT_RECORDS   OUTPUT_RECORDS
- ----------------------------------------------------------------------------------------------
-      Map 1          16546.00        130,790          1,748      13,963,497           82,778
-     Map 10           4568.00         49,540            484      27,755,681           15,784
-     Map 12           3554.00         75,240            987      55,261,069           41,887
-      Map 6          14520.00        131,700          5,584       6,000,000           42,697
-      Map 7          13490.00         34,760            780       1,920,800        1,920,800
-      Map 8           4073.00         80,900            472     106,067,119           73,854
-      Map 9           3146.00         10,200            375          10,000              366
- Reducer 11           1531.00         27,450            623          15,784          157,859
- Reducer 13           1026.00         32,250            697          41,887          261,474
-  Reducer 2           4577.00        267,060          4,652         575,965           22,170
-  Reducer 3           4046.00        263,850          5,876          22,170           61,923
-  Reducer 4            503.00          1,430             20          61,923                0
-  Reducer 5          12877.00          2,360              0          82,778                3
- ----------------------------------------------------------------------------------------------

+------------+--------------------+-----------------------+-------+-----------------------+-------+-------------------+-------+---------------+-------+------------------------+-------+-----------------------+-------+
| cd_gender  | cd_marital_status  |  cd_education_status  | cnt1  | cd_purchase_estimate  | cnt2  | cd_credit_rating  | cnt3  | cd_dep_count  | cnt4  | cd_dep_employed_count  | cnt5  | cd_dep_college_count  | cnt6  |
+------------+--------------------+-----------------------+-------+-----------------------+-------+-------------------+-------+---------------+-------+------------------------+-------+-----------------------+-------+
| F          | D                  | 2 yr Degree           | 1     | 500                   | 1     | Good              | 1     | 0             | 1     | 0                      | 1     | 4                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 500                   | 1     | Good              | 1     | 1             | 1     | 0                      | 1     | 5                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 500                   | 1     | Good              | 1     | 1             | 1     | 1                      | 1     | 1                     | 1     |
....
| F          | D                  | 2 yr Degree           | 1     | 3500                  | 1     | High Risk         | 1     | 0             | 1     | 4                      | 1     | 4                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 3500                  | 1     | High Risk         | 1     | 3             | 1     | 2                      | 1     | 1                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 3500                  | 1     | High Risk         | 1     | 5             | 1     | 4                      | 1     | 0                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 3500                  | 1     | High Risk         | 1     | 5             | 1     | 6                      | 1     | 6                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 3500                  | 1     | Low Risk          | 1     | 0             | 1     | 3                      | 1     | 4                     | 1     |
+------------+--------------------+-----------------------+-------+-----------------------+-------+-------------------+-------+---------------+-------+------------------------+-------+-----------------------+-------+
100 rows selected (61.765 seconds)
```
tez引擎 + llap 执行情况；
```xml
INFO  : Query ID = sywu01_20201023174916_f4c7d891-7395-4720-9eb1-5bf1fd7c024c
INFO  : Total jobs = 1
INFO  : Launching Job 1 out of 1
INFO  : Starting task [Stage-1:MAPRED] in parallel
----------------------------------------------------------------------------------------------
        VERTICES      MODE        STATUS  TOTAL  COMPLETED  RUNNING  PENDING  FAILED  KILLED  
----------------------------------------------------------------------------------------------
Map 6 ..........      llap     SUCCEEDED      3          3        0        0       0       0  
Map 7 ..........      llap     SUCCEEDED      4          4        0        0       0       0  
Map 1 ..........      llap     SUCCEEDED      6          6        0        0       0       0  
Map 9 ..........      llap     SUCCEEDED      1          1        0        0       0       0  
Reducer 5 ......      llap     SUCCEEDED      1          1        0        0       0       0  
Map 8 ..........      llap     SUCCEEDED      6          6        0        0       0       0  
Map 10 .........      llap     SUCCEEDED      6          6        0        0       0       0  
Reducer 11 .....      llap     SUCCEEDED    234        234        0        0       0       0  
Map 12 .........      llap     SUCCEEDED      7          7        0        0       0       0  
Reducer 13 .....      llap     SUCCEEDED    234        234        0        0       0       0  
Reducer 2 ......      llap     SUCCEEDED    234        234        0        0       0       2  
Reducer 3 ......      llap     SUCCEEDED    145        145        0        0       0       3  
Reducer 4 ......      llap     SUCCEEDED      1          1        0        0       0       0  
----------------------------------------------------------------------------------------------
VERTICES: 13/13  [==========================>>] 100%  ELAPSED TIME: 11.34 s    
----------------------------------------------------------------------------------------------
INFO  : Completed executing command(queryId=sywu01_20201023174916_f4c7d891-7395-4720-9eb1-5bf1fd7c024c); Time taken: 30.035 seconds
INFO  : OK

 - Query Execution Summary
 - ----------------------------------------------------------------------------------------------
 - OPERATION                            DURATION
 - ----------------------------------------------------------------------------------------------
 - Compile Query                           0.00s
 - Prepare Plan                            0.00s
 - Get Query Coordinator (AM)              0.00s
 - Submit Plan                         1603446581.98s
 - Start DAG                               1.00s
 - Run DAG                                11.02s
 - ----------------------------------------------------------------------------------------------
 - 
 - Task Execution Summary
 - ----------------------------------------------------------------------------------------------
 -   VERTICES      DURATION(ms)   CPU_TIME(ms)    GC_TIME(ms)   INPUT_RECORDS   OUTPUT_RECORDS
 - ----------------------------------------------------------------------------------------------
 -      Map 1           2042.00              0              0      13,963,497           82,778
 -     Map 10           1527.00              0              0      27,755,681           15,767
 -     Map 12           1530.00              0              0      55,261,069           40,935
 -      Map 6            514.00              0              0       6,000,000           42,697
 -      Map 7            514.00              0              0       1,920,800        1,920,800
 -      Map 8           3063.00              0              0     106,067,119           59,399
 -      Map 9              0.00              0              0          10,000              366
 - Reducer 11           1536.00              0              0          15,767           14,579
 - Reducer 13           1019.00              0              0          40,935           33,694
 -  Reducer 2           3059.00              0              0         190,450           22,170
 -  Reducer 3           2540.00              0              0          22,170           14,423
 -  Reducer 4            278.00              0              0          14,423                0
 -  Reducer 5           2041.00              0              0          82,778                3
 - ----------------------------------------------------------------------------------------------
 
+------------+--------------------+-----------------------+-------+-----------------------+-------+-------------------+-------+---------------+-------+------------------------+-------+-----------------------+-------+
| cd_gender  | cd_marital_status  |  cd_education_status  | cnt1  | cd_purchase_estimate  | cnt2  | cd_credit_rating  | cnt3  | cd_dep_count  | cnt4  | cd_dep_employed_count  | cnt5  | cd_dep_college_count  | cnt6  |
+------------+--------------------+-----------------------+-------+-----------------------+-------+-------------------+-------+---------------+-------+------------------------+-------+-----------------------+-------+
| F          | D                  | 2 yr Degree           | 1     | 500                   | 1     | Good              | 1     | 0             | 1     | 0                      | 1     | 4                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 500                   | 1     | Good              | 1     | 1             | 1     | 0                      | 1     | 5                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 500                   | 1     | Good              | 1     | 1             | 1     | 1                      | 1     | 1                     | 1     |
....
| F          | D                  | 2 yr Degree           | 1     | 3500                  | 1     | High Risk         | 1     | 0             | 1     | 4                      | 1     | 4                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 3500                  | 1     | High Risk         | 1     | 3             | 1     | 2                      | 1     | 1                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 3500                  | 1     | High Risk         | 1     | 5             | 1     | 4                      | 1     | 0                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 3500                  | 1     | High Risk         | 1     | 5             | 1     | 6                      | 1     | 6                     | 1     |
| F          | D                  | 2 yr Degree           | 1     | 3500                  | 1     | Low Risk          | 1     | 0             | 1     | 3                      | 1     | 4                     | 1     |
+------------+--------------------+-----------------------+-------+-----------------------+-------+-------------------+-------+---------------+-------+------------------------+-------+-----------------------+-------+
100 rows selected (37.668 seconds) 
```

## 5 总结
可以看到，***mr引擎的执行耗时(850.686 seconds)是tez引擎执行耗时(61.765 seconds)和tez引擎+llap执行耗时(37.668 seconds)的近14倍***，资源使用率远远高于后者；tez引擎和tez引擎+llap确实极大的提升了查询性能，也让hive更越进一步，而这一切的代价，仅是对架构、底层的了解和认识以及组件的升级和更新能够获得的。

## 链接
* https://cwiki.apache.org/confluence/display/Hive/LLAP - Hive llap
* https://issues.apache.org/jira/browse/YARN-4692 - Simplified and first-class support for services in YARN
* http://hadoop.apache.org/docs/r3.1.0 - Apache Hadoop 3.1.0
* http://incubator.apache.org/projects/slider.html - Apache slider
* https://issues.apache.org/jira/browse/HIVE-9850 - documentation for llap
* https://issues.apache.org/jira/browse/HIVE-7926 - long-lived daemons for query fragment execution, I/O and caching 
* https://github.com/hortonworks/hive-testbench - A testbench for experimenting with Apache Hive at any data scale.