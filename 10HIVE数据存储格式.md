>
>
> Hive支持的存储数的格式主要有：TEXTFILE（行式存储） 、SEQUENCEFILE(行式存储)、ORC（列式存储）、PARQUET（列式存储）。

# 列式存储和行式存储

![1620999146160](./assets\1620999146160.png)

- **行存储的特点：** 查询满足条件的一整行数据的时候，列存储则需要去每个聚集的字段找到对应的每个列的值，行存储只需要找到其中一个值，其余的值都在相邻地方，所以此时行存储查询的速度更快。

- **列存储的特点：** 因为每个字段的数据聚集存储，在查询只需要少数几个字段的时候，能大大减少读取的数据量；每个字段的数据类型一定是相同的，列式存储可以针对性的设计更好的设计压缩算法。

**相比于行式存储，列式存储在分析场景下有着许多优良的特性****:**

1. 分析场景中往往需要读大量行但是少数几个列。在行存模式下，数据按行连续存储，所有列的数据都存储在一个block中，不参与计算的列在IO时也要全部读出，读取操作被严重放大。而列存模式下，只需要读取参与计算的列即可，极大的减低了IO开销，加速了查询。

2. 同一列中的数据属于同一类型，压缩效果显著。列存储往往有着高达十倍甚至更高的压缩比，节省了大量的存储空间，降低了存储成本。

3. 更高的压缩比意味着更小的数据空间，从磁盘中读取相应数据耗时更短。

4. 自由的压缩算法选择。不同列的数据具有不同的数据类型，适用的压缩算法也就不尽相同。可以针对不同列类型，选择最合适的压缩算法。

**TEXTFILE和SEQUENCEFILE的存储格式都是基于行存储的；**

**ORC和PARQUET是基于列式存储的。**

#  **TEXTFILE格式**

> 默认格式，数据不做压缩，磁盘开销大，数据解析开销大。可结合Gzip、Bzip2使用(系统自动检查，执行查询时自动解压)，但使用这种方式，hive不会对数据进行切分，从而无法对数据进行并行操作。

# ORC格式

> ORC的全称是(Optimized Record Columnar)，使用ORC文件格式可以提高hive读、写和处理数据的能力。
>
> 在ORC格式的hive表中，记录首先会被横向的切分为多个stripes，然后在每一个stripe内数据以列为单位进行存储，所有列的内容都保存在同一个文件中。每个stripe的默认大小为256MB，相对于RCFile每个4MB的stripe而言，更大的stripe使ORC的数据读取更加高效。 

![1620999305469](./assets\1620999305469.png)

ORC在RCFile的基础上进行了一定的改进，所以与RCFile相比，具有以下一些优势： 

1. ORC中的特定的序列化与反序列化操作可以使ORC file writer根据数据类型进行写出。 

2. 提供了多种RCFile中没有的indexes，这些indexes可以使ORC的reader很快的读到需要的数据，并且跳过无用数据，这使得ORC文件中的数据可以很快的得到访问。 

3. 由于ORC file writer可以根据数据类型进行写出，所以ORC可以支持复杂的数据结构（比如Map等）。 

4. 除了上面三个理论上就具有的优势之外，ORC的具体实现上还有一些其他的优势，比如ORC的stripe默认大小更大，为ORC writer提供了一个memory manager来管理内存使用情况。 

# PARQUET格式

> Parquet是面向分析型业务的列式存储格式，由Twitter和Cloudera合作开发，2015年5月从Apache的孵化器里毕业成为Apache顶级项目。
>
> Parquet文件是以二进制方式存储的，所以是不可以直接读取的，文件中包括该文件的数据和元数据，因此Parquet格式文件是自解析的。
>
> Parquet 在同一个数据文件中保存一行中的所有数据，以确保在同一个节点上处理时一行的所有列都可用。Parquet 所做的是设置 HDFS 块大小和最大数据文件大小为 1GB，以确保 I/O 和网络传输请求适用于大批量数据。



![img](./assets\wps2.jpg)

Parquet文件在磁盘所有数据分成多个RowGroup 和 Footer。

1. RowGroup: 真正存储数据区域，每一个RowGroup存储多个ColumnChunk的数据。

2. ColumnChunk就代表当前RowGroup某一列的数据，因为可能这一列还在其他RowGroup有数据。ColumnChunk可能包含一个Page。

3. Page是压缩和编码的单元，主要包括PageHeader，RepetitionLevel,DefinitionLevel和Values.

4. PageHeader： 包含一些元数据，诸如编码和压缩类型，有多少数据，当前page第一个数据的偏移量，当前Page第一个索引的偏移量，压缩和解压的大小

5. DefinitionLevel: 当前字段在路径中的深度

6. RepetitionLevel: 当前字段是否可以重复

7. Footer:主要当前文件的元数据和一些统计信息

#  主流文件存储格式对比实验

> 从存储文件的压缩比和查询速度两个角度对比。
>
> 存储文件的压缩比测试：

## TextFile

（1）创建表，存储数据格式为TEXTFILE

```sql
create table log_text (
track_time string,
url string,
session_id string,
referer string,
ip string,
end_user_id string,
city_id string
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE ;
```

(3)向表中加载数据

```sql
load data local inpath '/export/test/log.data' into table log_text ;
```

(4) 查看表中数据大小

```sql
dfs -du -h /user/hive/warehouse/myhive.db/log_text;
```

## **ORC**

- 创建表，存储数据格式为ORC

```sql
create table log_orc(
track_time string,
url string,
session_id string,
referer string,
ip string,
end_user_id string,
city_id string
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
STORED AS orc ;
```

(2) 向表中加载数据

```sql
insert into table log_orc select * from log_text ;
```

(3) 查看表中数据大小

```sql
dfs -du -h /user/hive/warehouse/myhive.db/log_orc;
```

## **Parquet**

（1）创建表，存储数据格式为parquet

```sql
create table log_parquet(
track_time string,
url string,
session_id string,
referer string,
ip string,
end_user_id string,
city_id string
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
STORED AS PARQUET ;
```

（2）向表中加载数据

```sql
insert into table log_parquet select * from log_text ;
```

(3) 查看表中数据大小

```sql
dfs -du -h /user/hive/warehouse/myhive.db/log_parquet;
```

## **存储文件的压缩比总结：**

**ORC** **>**  **Parquet >**  **textFile**

# **存储文件的查询速度测试：**

```sql
1）TextFile
hive (default)> select count(*) from log_text;
_c0
100000
Time taken: 21.54 seconds, Fetched: 1 row(s)
2）ORC
hive (default)> select count(*) from log_orc;
_c0
100000
Time taken: 20.867 seconds, Fetched: 1 row(s)
3）Parquet
hive (default)> select count(*) from log_parquet;
_c0
100000
Time taken: 22.922 seconds, Fetched: 1 row(s)

```

## **存储文件的查询速度总结：**

​	**ORC > TextFile > Parquet**

# 存储和压缩结合

ORC存储方式的压缩：

| Key                      | Default    | Notes                                                        |
| ------------------------ | ---------- | ------------------------------------------------------------ |
| orc.compress             | ZLIB       | high level compression (one of NONE, ZLIB, SNAPPY)           |
| orc.compress.size        | 262,144    | number of bytes in each compression chunk                    |
| orc.stripe.size          | 67,108,864 | number of bytes in each stripe                               |
| orc.row.index.stride     | 10,000     | number of rows between index entries (must be >= 1000)       |
| orc.create.index         | true       | whether to create row indexes                                |
| orc.bloom.filter.columns | ""         | comma separated list of column names for which bloom filter should be created |
| orc.bloom.filter.fpp     | 0.05       | false positive probability for bloom filter (must >0.0 and <1.0) |

##  **创建一个非压缩的的ORC存储方式**

- 建表语句

~~~sql
create table log_orc_none(
track_time string,
url string,
session_id string,
referer string,
ip string,
end_user_id string,
city_id string
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
STORED AS orc tblproperties ("orc.compress"="NONE");
~~~

- 插入数据

```sql
insert into table log_orc_none select * from log_text ;
```

- 查看插入后数据

```sql
dfs -du -h /user/hive/warehouse/myhive.db/log_orc_none;
```

## 创建一个SNAPPY压缩的ORC存储方式

- 建表语句

~~~sql
create table log_orc_snappy(
track_time string,
url string,
session_id string,
referer string,
ip string,
end_user_id string,
city_id string
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
STORED AS orc tblproperties ("orc.compress"="SNAPPY");
~~~



- 插入数据

```sql
insert into table log_orc_snappy select * from log_text ;
```

- 查看插入后数据

~~~sql
dfs -du -h /user/hive/warehouse/myhive.db/log_orc_snappy ;
~~~

# **存储方式和压缩总结：**

​	在实际的项目开发当中，hive表的数据存储格式一般选择：orc或parquet。压缩方式一般选择snappy。