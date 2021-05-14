# HAVE的介绍

- ive是一个构建在Hadoop上的数据仓库框架。最初，Hive是由Facebook开发，后来移交由Apache软件基金会开发，并作为一个Apache开源项目。
- Hive是基于Hadoop的一个数据仓库工具，可以将结构化的数据文件映射为一张数据库表，并提供类SQL查询功能。
- 其本质是将SQL转换为MapReduce的任务进行运算，底层由HDFS来提供数据的存储，说白了hive可以理解为一个将SQL转换为MapReduce的任务的工具，甚至更进一步可以说hive就是一个MapReduce的客户端。

![1620823439840](./assets\1620823439840.png)

# 为什么使用HAVE

- 直接使用Hadoop所面临的问题
  - 人员学习成本高
  - 项目周期要求太短
  - MapReduce实现复杂查询逻辑开发难度太大 
- 为什么用HAVE
  - 操作接口采用类SQL语法，提供快速开发的能力
  - 避免了去写MapReduce，减少开发人员的学习成本
  - 功能扩展很方便

# HIVE的特点

1. Hive最大的特点是通过类SQL来分析大数据，而避免了写MapReduce程序来分析数据，这样使得分析数据更容易。

2. 数据是存储在HDFS上的，Hive本身并不提供数据的存储功能，它可以使已经存储的数据结构化。

3. Hive是将数据映射成数据库和一张张的表，库和表的元数据信息一般存在关系型数据库上（比如MySQL）。

4. 数据存储方面：它能够存储很大的数据集，可以直接访问存储在Apache HDFS或其他数据存储系统（如Apache HBase）中的文件。

5. 数据处理方面：因为Hive语句最终会生成MapReduce任务去计算，所以不适用于实时计算的场景，它适用于离线分析。

6. Hive除了支持MapReduce计算引擎，还支持Spark和Tez这两种分布式计算引擎；

7. 数据的存储格式有多种，比如数据源是二进制格式，普通文本格式等等；

# Hive架构

![1620824106231](./assets\1620824106231.png)

## **基本组成**

- **客户端:**
  - Client CLI(hive shell 命令行),JDBC/ODBC(java访问hive),WEBUI(浏览器访问hive)

- **元数据:**
  - Metastore:元数据包括:表名,表所属数据库(默认是default) ,表的拥有者,列/分区字段,表的类型(是否是外部表),表的数据所在目录等.默认存储在自带的derby数据库中,推荐使用MySQL存储Metastore

- **驱动器:Driver**

1. 解析器(SQL Parser):将SQL字符转换成抽象语法树AST,这一步一般使用都是第三方工具库完成,比如antlr,对AST进行语法分析,比如表是否存在,字段是否存在,SQL语句是否有误

2. 编译器(Physical Plan):将AST编译生成逻辑执行计划
3. 优化器(Query Optimizer):对逻辑执行计划进行优化

4. 执行器(Execution):把逻辑执行计划转换成可以运行的物理计划,对于Hive来说,就是MR/Spark

- **存储和执行：**
  - Hive使用HDFS进行存储,使用MapReduce进行计算

## Hive与传统数据库对比

![1620824429921](./assets\1620824429921.png)

**总结：hive具有sql数据库的外表，但应用场景完全不同，hive只适合用来做批量数据统计分析**

# HAVE的元数据

**元数据**(metadata)：本质上只是用来存储hive中有哪些数据库，哪些表，表的字段，分区，索引以及命名空间等元信息。元数据存储在关系型数据库中。如hive内置的Derby、第三方数据库如MySQL等。

**元数据服务(metastore）**，作用是：客户端连接metastore服务，metastore再去连接MySQL数据库来存取元数据。有了metastore服务，就可以有多个客户端同时连接，而且这些客户端不需要知道MySQL数据库的用户名和密码，只需要连接metastore 服务即可。