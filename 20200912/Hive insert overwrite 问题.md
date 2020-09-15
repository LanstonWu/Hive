Hive insert overwrite 问题
====

>微信公众号：苏言论  
理论联系实际，畅言技术与生活。

## 1 测试的版本
Apache hive 1.1.0/2.3.1/3.1.0

## 2 insert overwrite使用说明
表类型|使用场景|hive操作逻辑
:---:|:---|:---
非分区表|insert overwrite table t select col from b|先删除现有数据表目录，再插入新的数据，不存在旧数据保留的情况
分区表|insert overwrite table t partition(part_name='') select col from b|先删除现有分区数据目录，再插入新的数据，不存在旧数据保留的情况
分区表|insert overwrite table t select col,part_name from b|只对新数据集中存在数据变更的分区进行覆盖，不涉及数据变更的分区和旧数据会被保留

## 3 示例
考虑下面的课程安排表数据结构和数据；
```sql
drop table class_course_schedule;
create table class_course_schedule(id int,course_name string,course_time date) partitioned by(city string);
insert into class_course_schedule values
(1,'语文','2020-02-01','guangzhou'),
(2,'语文','2020-02-02','guangzhou'),
(10,'语文','2020-02-03','guangzhou'),
(3,'政治','2020-02-01','hangzhou'),
(4,'体育','2020-02-02','hangzhou'),
(5,'英语','2020-02-03','hangzhou'),
(6,'音乐','2020-02-01','shanghai'),
(7,'物理','2020-02-02','shanghai'),
(8,'计算机','2020-02-03','shanghai'),
(9,'画图','2020-02-04','shanghai');

select * from class_course_schedule t;
+-------+----------------+----------------+------------+
| t.id  | t.course_name  | t.course_time  |   t.city   |
+-------+----------------+----------------+------------+
| 1     | 语文           | 2020-02-01     | guangzhou  |
| 2     | 语文           | 2020-02-02     | guangzhou  |
| 10    | 语文           | 2020-02-03     | guangzhou  |
| 3     | 政治           | 2020-02-01     | hangzhou   |
| 4     | 体育           | 2020-02-02     | hangzhou   |
| 5     | 英语           | 2020-02-03     | hangzhou   |
| 6     | 音乐           | 2020-02-01     | shanghai   |
| 7     | 物理           | 2020-02-02     | shanghai   |
| 8     | 计算机         | 2020-02-03     | shanghai   |
| 9     | 画图           | 2020-02-04     | shanghai   |
+-------+----------------+----------------+------------+
10 rows selected (1.459 seconds)
```
对表进行insert overwrite，比如；
```sql
insert overwrite table class_course_schedule select * from class_course_schedule where course_name!='语文';
```
由于city='guangzhou'只有course_name='语文'的数据记录，select返回的数据集中city='guangzhou'分区的数据没有变化，hive从返回的select数据集中删除整个city='guangzhou'分区的情况下进行过滤，所以旧分区(city='guangzhou')没有被重写，数据被保留；
```sql
select * from class_course_schedule t;
+-------+----------------+----------------+------------+
| t.id  | t.course_name  | t.course_time  |   t.city   |
+-------+----------------+----------------+------------+
| 1     | 语文           | 2020-02-01     | guangzhou  |
| 2     | 语文           | 2020-02-02     | guangzhou  |
| 10    | 语文           | 2020-02-03     | guangzhou  |
| 3     | 政治           | 2020-02-01     | hangzhou   |
| 4     | 体育           | 2020-02-02     | hangzhou   |
| 5     | 英语           | 2020-02-03     | hangzhou   |
| 6     | 音乐           | 2020-02-01     | shanghai   |
| 7     | 物理           | 2020-02-02     | shanghai   |
| 8     | 计算机         | 2020-02-03     | shanghai   |
| 9     | 画图           | 2020-02-04     | shanghai   |
+-------+----------------+----------------+------------+
10 rows selected (1.459 seconds)
```
***hive按设计工作，因为只需要覆盖所需分区的情况，对于增量分区负载来说是正常的，在这种情况下，无需触摸其它分区；如果覆盖无需更改的分区，则会导致非必要数据丢失，恢复起来可能会非常昂贵***。  
再对表进行insert overwrite；
```sql
insert overwrite table class_course_schedule select * from class_course_schedule where course_name!='英语';
```
由于 course_name='英语'所在的分区city='hangzhou'中course_name非一致数据，select返回的数据集中city='guangzhou'分区的数据有变化，hive对其重写；
```sql
+-------+----------------+----------------+------------+
| t.id  | t.course_name  | t.course_time  |   t.city   |
+-------+----------------+----------------+------------+
| 1     | 语文           | 2020-02-01     | guangzhou  |
| 2     | 语文           | 2020-02-02     | guangzhou  |
| 10    | 语文           | 2020-02-03     | guangzhou  |
| 3     | 政治           | 2020-02-01     | hangzhou   |
| 4     | 体育           | 2020-02-02     | hangzhou   |
| 6     | 音乐           | 2020-02-01     | shanghai   |
| 7     | 物理           | 2020-02-02     | shanghai   |
| 8     | 计算机         | 2020-02-03     | shanghai   |
| 9     | 画图           | 2020-02-04     | shanghai   |
+-------+----------------+----------------+------------+
9 rows selected (0.562 seconds)
```

## 4 建议的操作
1. 分区表指定具体分区，减少对不相关分区的影响；

```sql
insert overwrite table class_course_schedule partition(city='guangzhou') values
(12,'语文01','2020-02-12'),
(22,'语文02','2020-02-22'),
(19,'语文03','2020-02-19');

+-------+----------------+----------------+------------+
| t.id  | t.course_name  | t.course_time  |   t.city   |
+-------+----------------+----------------+------------+
| 12    | 语文01         | 2020-02-12     | guangzhou  |
| 22    | 语文02         | 2020-02-22     | guangzhou  |
| 19    | 语文03         | 2020-02-19     | guangzhou  |
| 3     | 政治           | 2020-02-01     | hangzhou   |
| 4     | 体育           | 2020-02-02     | hangzhou   |
| 6     | 音乐           | 2020-02-01     | shanghai   |
| 7     | 物理           | 2020-02-02     | shanghai   |
| 8     | 计算机         | 2020-02-03     | shanghai   |
| 9     | 画图           | 2020-02-04     | shanghai   |
+-------+----------------+----------------+------------+
9 rows selected (0.574 seconds)
```

2. 如果仍然要删除分区，可以删除/重建表(可能需要为此再创建一个中间表)，然后将分区加载到其中；或者计算需要单独删除的分区，然后执行ALTER TABLE DROP PARTITION删除分区。

## 5 参考链接
[Inserting datain to HiveTables from queries](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DML#LanguageManualDML-InsertingdataintoHiveTablesfromqueries)