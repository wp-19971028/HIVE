# hive调优

## Fetch抓取（Hive可以避免进MapReduce）

> Hive中对某些情况的查询可以不必使用MapReduce计算。例如：SELECT * FROM employees;在这种情况下，Hive可以简单地读取employee对应的存储目录下的文件，然后输出查询结果到控制台。

> 在hive-default.xml.template文件中hive.fetch.task.conversion默认是more，老版本hive默认是minimal，该属性修改为more以后，在全局查找、字段查找、limit查找等都不走mapreduce。

案例实操：

1. 把hive.fetch.task.conversion设置成none，然后执行查询语句，都会执行mapreduce程序。

```sql
hive (default)> set hive.fetch.task.conversion=none;
hive (default)> select * from score;
hive (default)> select s_score from score;
hive (default)> select s_score from score limit 3;
```

2. 把hive.fetch.task.conversion设置成more，然后执行查询语句，如下查询方式都不会执行mapreduce程序。

```sql
hive (default)> set hive.fetch.task.conversion=more;
hive (default)> select * from score;
hive (default)> select s_score from score;
hive (default)> select s_score from score limit 3;
```

## 本地模式

大多数的Hadoop Job是需要Hadoop提供的完整的可扩展性来处理大数据集的。不过，有时Hive的输入数据量是非常小的。在这种情况下，为查询触发执行任务时消耗可能会比实际job的执行时间要多的多。对于大多数这种情况，Hive可以通过本地模式在单台机器上处理所有的任务。对于小数据集，执行时间可以明显被缩短。

用户可以通过设置hive.exec.mode.local.auto的值为true，来让Hive在适当的时候自动启动这个优化。

```sql
set hive.exec.mode.local.auto=true;  --开启本地mr
--设置local mr的最大输入数据量，当输入数据量小于这个值时采用local  mr的方式，默认为134217728，即128M
set hive.exec.mode.local.auto.inputbytes.max=51234560;
--设置local mr的最大输入文件个数，当输入文件个数小于这个值时采用local mr的方式，默认为4
set hive.exec.mode.local.auto.input.files.max=10;
```

案例实操：

~~~sql
--1）开启本地模式，并执行查询语句
hive (default)> set hive.exec.mode.local.auto=true; 
hive (default)> select * from score cluster by s_id;
18 rows selected (1.568 seconds)

--2）关闭本地模式，并执行查询语句
hive (default)> set hive.exec.mode.local.auto=false; 
hive (default)> select * from score cluster by s_id;
18 rows selected (11.865 seconds)
~~~

## Join的优化

如果不指定MapJoin或者不符合MapJoin的条件，那么Hive解析器会将Join操作转换成Common Join，即：在Reduce阶段完成join。容易发生数据倾斜。可以用MapJoin把小表全部加载到内存在map端进行join，避免reducer处理

![1621006253166](./assets\1621006253166.png)

首先是Task A，它是一个Local Task（在客户端本地执行的Task），负责扫描小表b的数据，将其转换成一个HashTable的数据结构，并写入本地的文件中，之后将该文件加载到DistributeCache中。

接下来是Task B，该任务是一个没有Reduce的MR，启动MapTasks扫描大表a,在Map阶段，根据a的每一条记录去和DistributeCache中b表对应的HashTable关联，并直接输出结果。

由于MapJoin没有Reduce，所以由Map直接输出结果文件，有多少个Map Task，就有多少个结果文件。

**map端join**的参数设置：

```sql
-- （1）设置自动选择mapjoin
set hive.auto.convert.join = true; -- 默认为true
--  (2）大表小表的阈值设置：
set hive.mapjoin.smalltable.filesize= 25000000;
```

小表的输入文件大小的阈值（以字节为单位）;如果文件大小小于此阈值，它将尝试将common join转换为map join。

因此在实际使用中，只要根据业务把握住小表的阈值标准即可，hive会自动帮我们完成mapjoin，提高执行的效率。

## 大表Join大表

### 空KEY过滤

> 有时join超时是因为某些key对应的数据太多，而相同key对应的数据都会发送到相同的reducer上，从而导致内存不够。此时我们应该仔细分析这些异常的key，很多情况下，这些key对应的数据是异常数据，我们需要在SQL语句中进行过滤。例如key对应的字段为空，操作如下：
>
> 环境准备：

~~~sql
create table ori(id bigint, time bigint, uid string, keyword string, url_rank int, click_num int, click_url string) row format delimited fields terminated by '\t';

create table nullidtable(id bigint, time bigint, uid string, keyword string, url_rank int, click_num int, click_url string) row format delimited fields terminated by '\t';

create table jointable(id bigint, time bigint, uid string, keyword string, url_rank int, click_num int, click_url string) row format delimited fields terminated by '\t';

load data local inpath '/export/server/hivedatas/hive_big_table/*' into table ori; 
load data local inpath '/export/server/hivedatas/hive_have_null_id/*' into table nullidtable;
~~~



**不过滤：**

```sql
INSERT OVERWRITE TABLE jointable
SELECT a.* FROM nullidtable a JOIN ori b ON a.id = b.id;
结果：
No rows affected (152.135 seconds)
```

**过滤**

```sql
INSERT OVERWRITE TABLE jointable
SELECT a.* FROM (SELECT * FROM nullidtable WHERE id IS NOT NULL ) a JOIN ori b ON a.id = b.id;
结果：
No rows affected (141.585 seconds)
```

### 空key转换

> 有时虽然某个key为空对应的数据很多，但是相应的数据不是异常数据，必须要包含在join的结果中，此时我们可以表a中key为空的字段赋一个随机的值，使得数据随机均匀地分不到不同的reducer上。例如：

**不随机分布：**

```sql
set hive.exec.reducers.bytes.per.reducer=32123456;
set mapreduce.job.reduces=7;
INSERT OVERWRITE TABLE jointable
SELECT a.*
FROM nullidtable a
LEFT JOIN ori b ON CASE WHEN a.id IS NULL THEN 'hive' ELSE a.id END = b.id;

No rows affected (41.668 seconds)   52.477
```

> 结果：这样的后果就是所有为null值的id全部都变成了相同的字符串，及其容易造成数据的倾斜（所有的key相同，相同key的数据会到同一个reduce当中去）
>
>  
>
> 为了解决这种情况，我们可以通过hive的rand函数，随记的给每一个为空的id赋上一个随机值，这样就不会造成数据倾斜

**随机分布：**

```sql
set hive.exec.reducers.bytes.per.reducer=32123456;
set mapreduce.job.reduces=7;
INSERT OVERWRITE TABLE jointable
SELECT a.*
FROM nullidtable a
LEFT JOIN ori b ON CASE WHEN a.id IS NULL THEN concat('hive', rand()) ELSE a.id END = b.id;


No rows affected (42.594 seconds)
```

##### 案例实操

1. 需求：测试大表JOIN小表和小表JOIN大表的效率 （新的版本当中已经没有区别了，旧的版本当中需要使用小表）

2. 建大表、小表和JOIN后表的语句

```sql
create table bigtable(id bigint, time bigint, uid string, keyword string, url_rank int, click_num int, click_url string) row format delimited fields terminated by '\t';

create table smalltable(id bigint, time bigint, uid string, keyword string, url_rank int, click_num int, click_url string) row format delimited fields terminated by '\t';

create table jointable2(id bigint, time bigint, uid string, keyword string, url_rank int, click_num int, click_url string) row format delimited fields terminated by '\t';
```

3. 分别向大表和小表中导入数据

```sql
load data local inpath '/export/server/hivedatas/big_data' into table bigtable;
load data local inpath '/export/server/hivedatas/small_data' into table smalltable;
```

4. 关闭mapjoin功能（默认是打开的)

```sql
set hive.auto.convert.join = false;
```

5.  执行小表JOIN大表语句

````sql
INSERT OVERWRITE TABLE jointable2
SELECT b.id, b.time, b.uid, b.keyword, b.url_rank, b.click_num, b.click_url
FROM smalltable s
left JOIN bigtable  b
ON b.id = s.id;
-- Time taken: 67.411 seconds  
````

6. 执行大表JOIN小表语句

```sql
INSERT OVERWRITE TABLE jointable2
SELECT b.id, b.time, b.uid, b.keyword, b.url_rank, b.click_num, b.click_url
FROM bigtable  b
left JOIN smalltable  s
ON s.id = b.id;
-- Time taken: 69.376seconds
```

可以看出大表join小表或者小表join大表，就算是关闭map端join的情况下，在新的版本当中基本上没有区别了（hive为了解决数据倾斜的问题，会自动进行过滤）

## **SQL优化**

### **列裁剪**

Hive在读数据的时候，可以只读取查询中所需要用到的列，而忽略其他列。例如，若有以下查询：

```sql
SELECT a,b FROM q WHERE e<10;
```

在实施此项查询中，Q表有5列（a，b，c，d，e），Hive只读取查询逻辑中真实需要的3列a、b、e， 而忽略列c，d；这样做节省了读取开销，中间表存储开销和数据整合开销。

裁剪对应的参数项为：hive.optimize.cp=true（默认值为真）

### **分区裁剪**

可以在查询的过程中减少不必要的分区。例如，若有以下查询：

```sql
SELECT * FROM (SELECT a1, COUNT(1) FROM T GROUP BY a1) subq WHERE subq.prtn=100; --（多余分区）
SELECT * FROM T1 JOIN (SELECT * FROM T2) subq ON (T1.a1=subq.a2) WHERE subq.prtn=100;
```

> 查询语句若将"subq.prtn=100"条件放入子查询中更为高效，可以减少读入的分区数目。Hive自动执行这种裁剪优化。
>
> 　分区参数为：hive.optimize.pruner=true（默认值为真）

### **Group By**

> 默认情况下，Map阶段同一Key数据分发给一个reduce，当一个key数据过大时就倾斜了。
>
> 并不是所有的聚合操作都需要在Reduce端完成，很多聚合操作都可以先在Map端进行部分聚合，最后在Reduce端得出最终结果。
>
> 开启Map端聚合参数设置

```sql
--（1）是否在Map端进行聚合，默认为True
set hive.map.aggr = true;
--（2）在Map端进行聚合操作的条目数目
set hive.groupby.mapaggr.checkinterval = 100000;
--（3）有数据倾斜的时候进行负载均衡（默认是false）
set hive.groupby.skewindata = true;
```

> 当选项设定为 true，生成的查询计划会有两个MR Job。第一个MR Job中，Map的输出结果会随机分布到Reduce中，每个Reduce做部分聚合操作，并输出结果，这样处理的结果是相同的Group By Key有可能被分发到不同的Reduce中，从而达到负载均衡的目的；第二个MR Job再根据预处理的数据结果按照Group By Key分布到Reduce中（这个过程可以保证相同的Group By Key被分布到同一个Reduce中），最后完成最终的聚合操作。

### Count(distinct)

> 数据量小的时候无所谓，数据量大的情况下，由于COUNT DISTINCT操作需要用一个Reduce Task来完成，这一个Reduce需要处理的数据量太大，就会导致整个Job很难完成，一般COUNT DISTINCT使用先GROUP BY再COUNT的方式替换：
>
> **环境准备：**

```sql
create table bigtable(id bigint, time bigint, uid string, keyword string, url_rank int, click_num int, click_url string) row format delimited fields terminated by '\t';
load data local inpath '/home/admin/softwares/data/100万条大表数据（id除以10取整）/bigtable' into table bigtable;
```

测试:

 **方式1**

```sql
set hive.exec.reducers.bytes.per.reducer=32123456;
SELECT count(DISTINCT id) FROM bigtable;
结果：
c0
10000
Time taken: 35.49 seconds, Fetched: 1 row(s)
```

**方式2**

```sql
set hive.exec.reducers.bytes.per.reducer=32123456;
SELECT count(id) FROM (SELECT id FROM bigtable GROUP BY id) a;
结果：
Stage-Stage-1: Map: 1  Reduce: 4   Cumulative CPU: 13.07 sec   HDFS Read: 120749896 HDFS Write: 464 SUCCESS
Stage-Stage-2: Map: 3  Reduce: 1   Cumulative CPU: 5.14 sec   HDFS Read: 8987 HDFS Write: 7 SUCCESS
_c0
10000
Time taken: 51.202 seconds, Fetched: 1 row(s)
```

虽然会多用一个Job来完成，但在数据量大的情况下，这个绝对是值得的。

## 笛卡尔积

> 尽量避免笛卡尔积，即避免join的时候不加on条件，或者无效的on条件，Hive只能使用1个reducer来完成笛卡尔积。
>
>  

##  **动态分区调整**

> 往hive分区表中插入数据时，hive提供了一个动态分区功能，其可以基于查询参数的位置去推断分区的名称，从而建立分区。使用Hive的动态分区，需要进行相应的配置。

> Hive的动态分区是以第一个表的分区规则，来对应第二个表的分区规则，将第一个表的所有分区，全部拷贝到第二个表中来，第二个表在加载数据的时候，不需要指定分区了，直接用第一个表的分区即可

1. 开启动态分区功能（默认true，开启）

```sql
 set hive.exec.dynamic.partition=true;
```

2. 设置为非严格模式（动态分区的模式，默认strict，表示必须指定至少一个分区为静态分区，nonstrict模式表示允许所有的分区字段都可以使用动态分区。）

```sql
set hive.exec.dynamic.partition.mode=nonstrict;
```

3. 在所有执行MR的节点上，最大一共可以创建多少个动态分区。

```sql
set  hive.exec.max.dynamic.partitions=1000;
```

4. 在每个执行MR的节点上，最大可以创建多少个动态分区。该参数需要根据实际的数据来设定。

```sql
set hive.exec.max.dynamic.partitions.pernode=100
```

5. 整个MR Job中，最大可以创建多少个HDFS文件。

```sql
在linux系统当中，每个linux用户最多可以开启1024个进程，每一个进程最多可以打开2048个文件，即持有2048个文件句柄，下面这个值越大，就可以打开文件句柄越大
```

```sql
set hive.error.on.empty.partition=false;
```

## 案例操作

需求：将ori中的数据按照时间(如：20111231234568)，插入到目标表ori_partitioned的相应分区中。

1. 准备数据原表

```sql
create table ori_partitioned(id bigint, time bigint, uid string, keyword string, url_rank int, click_num int, click_url string) 
PARTITIONED BY (p_time bigint) 
row format delimited fields terminated by '\t';

load data local inpath '/export/server/hivedatas/small_data' into  table ori_partitioned partition (p_time='20111230000010');

load data local inpath '/export/server/hivedatas/small_data' into  table ori_partitioned partition (p_time='20111230000011');
```

2.  创建目标分区表



```sql
create table ori_partitioned_target(id bigint, time bigint, uid string, keyword string, url_rank int, click_num int, click_url string) PARTITIONED BY (p_time STRING) row format delimited fields terminated by '\t'
```

3. 向目标分区表加载数据

如果按照之前介绍的往指定一个分区中Insert数据，那么这个需求很不容易实现。这时候就需要使用动态分区来实现。

```sql
INSERT overwrite TABLE ori_partitioned_target PARTITION (p_time)
SELECT id, time, uid, keyword, url_rank, click_num, click_url, p_time
FROM ori_partitioned;
```

注意：在SELECT子句的最后几个字段，必须对应前面PARTITION (p_time)中指定的分区字段，包括顺序。



4. 查看分区

```sql
show partitions ori_partitioned_target; 
-- p_time=20111230000010 
-- p_time=20111230000011

```

## **Map数**

**1. 通常情况下，作业会通过input的目录产生一个或者多个map任务**

主要的决定因素有：input的文件总个数，input的文件大小，集群设置的文件块大小(目前为128M，可在hive中通过set dfs.block.size;命令查看到，该参数不能自定义修改)；

**2. 举例**

​	a)  假设input目录下有1个文件a，大小为780M，那么hadoop会将该文件a分隔成7个块（6个128m的块和1个12m的块），从而产生7个map数。

​	b) 假设input目录下有3个文件a，b，c大小分别为10m，20m，150m，那么hadoop会分隔成4个块（10m，20m，128m，22m），从而产生4个map数。即，如果文件大于块大小(128m)，那么会拆分，如果小于块大小，则把该文件当成一个块。

**3. 是不是map数越多越好？**

答案是否定的。如果一个任务有很多小文件（远远小于块大小128m），则每个小文件也会被当做一个块，用一个map任务来完成，而一个map任务启动和初始化的时间远远大于逻辑处理的时间，就会造成很大的资源浪费。而且，同时可执行的map数是受限的。

**4. 是不是保证每个map处理接近128m的文件块，就高枕无忧了？**

答案也是不一定。比如有一个127m的文件，正常会用一个map去完成，但这个文件只有一个或者两个小字段，却有几千万的记录，如果map处理的逻辑比较复杂，用一个map任务去做，肯定也比较耗时。

针对上面的问题3和4，我们需要采取两种方式来解决：即减少map数和增加map数；



### 减少Map数

```sql
--假设一个SQL任务：
select count(1) from tab_info where pt = '2020-07-04';

--该任务共有194个文件，其中很多是远远小于128M的小文件，总大小9G，正常执行会用194个map任务。
--Map总共消耗的计算资源：SLOTS_MILLIS_MAPS= 623,020

--通过以下方法来在map执行前合并小文件，减少map数：
set mapred.max.split.size=112345600;
set mapred.min.split.size.per.node=112345600;
set mapred.min.split.size.per.rack=112345600;
set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;

--前面三个参数确定合并文件块的大小，大于文件块大小128m的，按照128m来分隔，
--小于128m,大于100m的，按照100m来分隔，把那些小于100m的（包括小文件和分隔大文件剩下的），
--进行合并,最终生成了74个块。

set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
--这个参数表示执行前进行小文件合并，

--再执行上面的语句，用了74个map任务，map消耗的计算资源：SLOTS_MILLIS_MAPS= 333,500
--对于这个简单SQL任务，执行时间上可能差不多，但节省了一半的计算资源。
```

### **增加Map数**

> 当input的文件都很大，任务逻辑复杂，map执行非常慢的时候，可以考虑增加Map数，来使得每个map处理的数据量减少，从而提高任务的执行效率。

```sql
--假设有这样一个任务：
select 
	data_desc,
	count(1),
	count(distinct id),
	sum(case when ...),
	sum(case when ...),
	sum(...)
from 
 a 
group by data_desc

/*
如果表a只有一个文件，大小为120M，但包含几千万的记录，如果用1个map去完成这个任务，肯定是比较耗时的，
这种情况下，我们要考虑将这一个文件合理的拆分成多个，
这样就可以用多个map任务去完成。
*/

set mapred.reduce.tasks=10;

create table a_1 as 
select * from tab_info  distribute by rand(123);

/*
这样会将a表的记录，随机的分散到包含10个文件的a_1表中，再用a_1代替上面sql中的a表，则会用10个map任务去完成。
每个map任务处理大于12M（几百万记录）的数据，效率肯定会好很多。
*/
```

> 这样会将a表的记录，随机的分散到包含10个文件的a_1表中，再用a_1代替上面sql中的a表，则会用10个map任务去完成。
>
>  
>
> 每个map任务处理大于12M（几百万记录）的数据，效率肯定会好很多。
>
> 看上去，貌似这两种有些矛盾，一个是要合并小文件，一个是要把大文件拆成小文件，这点正是重点需要关注的地方，根据实际情况，控制map数量需要遵循两个原则：使大数据量利用合适的map数；使单个map任务处理合适的数据量；



### Reduce数

> Reduce的个数对整个作业的运行性能有很大影响。如果Reduce设置的过大，那么将会产生很多小文件，对NameNode会产生一定的影响，而且整个作业的运行时间未必会减少；如果Reduce设置的过小，那么单个Reduce处理的数据将会加大，很可能会引起OOM异常。
>
> 如果设置了mapred.reduce.tasks/mapreduce.job.reduces参数，那么Hive会直接使用它的值作为Reduce的个数；如果mapred.reduce.tasks/mapreduce.job.reduces的值没有设置（也就是-1），那么Hive会根据输入文件的大小估算出Reduce的个数。根据输入文件估算Reduce的个数可能未必很准确，因为Reduce的输入是Map的输出，而Map的输出可能会比输入要小，所以最准确的数根据Map的输出估算Reduce的个数。

> **1. Hive自己如何确定reduce数：**
>
> 　　reduce个数的设定极大影响任务执行效率，不指定reduce个数的情况下，Hive会猜测确定一个reduce个数，基于以下两个设定：
>
> 　hive.exec.reducers.bytes.per.reducer（每个reduce任务处理的数据量，默认为1000^3=1G）
>
> 　hive.exec.reducers.max（每个任务最大的reduce数，默认为999）
>
> 　计算reducer数的公式很简单N=min（参数2，总输入数据量/参数1），即如果reduce的输入（map的输出）总大小不超过1G，那么只会有一个reduce任务；

```sql
select pt,count(1) from tab_info where pt = '2020-07-04' group by pt; 
-- 如果源文件总大小为9G多，那么这句会有10个reduce
```

**调整reduce个数方法一：**

调整hive.exec.reducers.bytes.per.reducer参数的值；

```sql
set hive.exec.reducers.bytes.per.reducer=524288000; --（500M）
select pt, count(1) from tab_info where pt = '2020-07-04' group by pt;
-- 如果源文件总大小为9G多,这次有20个reduce
```

 **调整reduce个数方法二：**

```sql
set mapred.reduce.tasks=15;
select pt,count(1) from tab_info where pt = '2020-07-04' group by pt;

-- 这次有15个reduce
```

**4. reduce个数并不是越多越好；**

　同map一样，启动和初始化reduce也会消耗时间和资源；

　另外，有多少个reduce，就会有个多少个输出文件，如果生成了很多个小文件，那么如果这些小文件作为下一个任务的输入，则也会出现小文件过多的问题；

**5. 什么情况下只有一个reduce；**

很多时候你会发现任务中不管数据量多大，不管你有没有调整reduce个数的参数，任务中一直都只有一个reduce任务；其实只有一个reduce任务的情况，除了数据量小于hive.exec.reducers.bytes.per.reducer参数值的情况外，还有以下原因：

- 没有group by的汇总，

比如

```sql
select pt,count(1) from tab_info where pt = ‘2020-07-04’ group by pt; 
-- 写成
select count(1) from tab_info where pt = ‘2020-07-04’; 
```

这点非常常见

- 用了Order by

- 有笛卡尔积。

　　注意：在设置reduce个数的时候也需要考虑这两个原则：使大数据量利用合适的reduce数；是单个reduce任务处理合适的数据量；

## 并行执行

> Hive会将一个查询转化成一个或者多个阶段。这样的阶段可以是MapReduce阶段、抽样阶段、合并阶段、limit阶段。或者Hive执行过程中可能需要的其他阶段。默认情况下，Hive一次只会执行一个阶段。不过，某个特定的job可能包含众多的阶段，而这些阶段可能并非完全互相依赖的，也就是说有些阶段是可以并行执行的，这样可能使得整个job的执行时间缩短。不过，如果有更多的阶段可以并行执行，那么job可能就越快完成。
>
> ​	通过设置参数hive.exec.parallel值为true，就可以开启并发执行。不过，在共享集群中，需要注意下，如果job中并行阶段增多，那么集群利用率就会增加。

```sql
set hive.exec.parallel=true;              --打开任务并行执行
set hive.exec.parallel.thread.number=16;  --同一个sql允许最大并行度，默认为8。

```

当然，得是在系统资源比较空闲的时候才有优势，否则，没资源，并行也起不来。

## 严格模式

Hive提供了一个严格模式，可以防止用户执行那些可能意想不到的不好的影响的查询。

通过设置属性hive.mapred.mode值为默认是非严格模式nonstrict 。开启严格模式需要修改hive.mapred.mode值为strict，开启严格模式可以禁止3种类型的查询。

```sql
set hive.mapred.mode = strict;  --开启严格模式
set hive.mapred.mode = nostrict; --开启非严格模式
```

**配置文件修改:**

```xml
<property>
    <name>hive.mapred.mode</name>
    <value>strict</value>
  </property>
```



1. 对于分区表，在where语句中必须含有分区字段作为过滤条件来限制范围，否则不允许执行。换句话说，就是用户不允许扫描所有分区。进行这个限制的原因是，通常分区表都拥有非常大的数据集，而且数据增加迅速。没有进行分区限制的查询可能会消耗令人不可接受的巨大资源来处理这个表。

 

2. 对于使用了order by语句的查询，要求必须使用limit语句。因为order by为了执行排序过程会将所有的结果数据分发到同一个Reducer中进行处理，强制要求用户增加这个LIMIT语句可以防止Reducer额外执行很长一段时间。

 

3. 限制笛卡尔积的查询。对关系型数据库非常了解的用户可能期望在执行JOIN查询的时候不使用ON语句而是使用where语句，这样关系数据库的执行优化器就可以高效地将WHERE语句转化成那个ON语句。不幸的是，Hive并不会执行这种优化，因此，如果表足够大，那么这个查询就会出现不可控的情况。



## 推测执行

在分布式集群环境下，因为程序Bug（包括Hadoop本身的bug），负载不均衡或者资源分布不均等原因，会造成同一个作业的多个任务之间运行速度不一致，有些任务的运行速度可能明显慢于其他任务（比如一个作业的某个任务进度只有50%，而其他所有任务已经运行完毕），则这些任务会拖慢作业的整体执行进度。为了避免这种情况发生，Hadoop采用了推测执行（Speculative Execution）机制，它根据一定的法则推测出“拖后腿”的任务，并为这样的任务启动一个备份任务，让该任务与原始任务同时处理同一份数据，并最终选用最先成功运行完成任务的计算结果作为最终结果。

设置开启推测执行参数：Hadoop的mapred-site.xml文件中进行配置

```sql
<property>
  <name>mapreduce.map.speculative</name>
  <value>true</value>
  <description>If true, then multiple instances of some map tasks 
               may be executed in parallel.</description>
</property>

<property>
  <name>mapreduce.reduce.speculative</name>
  <value>true</value>
  <description>If true, then multiple instances of some reduce tasks 
               may be executed in parallel.</description>
</property>
```

**hive**设置开启推测执行参数

```sql
set mapred.map.tasks.speculative.execution=true
set mapred.reduce.tasks.speculative.execution=true
set hive.mapred.reduce.tasks.speculative.execution=true;
```

关于调优这些推测执行变量，还很难给一个具体的建议。如果用户对于运行时的偏差非常敏感的话，那么可以将这些功能关闭掉。如果用户因为输入数据量很大而需要执行长时间的map或者Reduce task的话，那么启动推测执行造成的浪费是非常巨大大。

