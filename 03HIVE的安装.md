# HIVE的安装方式

HIVE的安装一共有三种方式: **内嵌模式、本地模式、远程模式**

## 内嵌模式

> **内嵌模式**使用的是内嵌的Derby数据库来存储元数据，也不需要额外起Metastore服务。数据库和Metastore服务都嵌入在主Hive Server进程中。这个是默认的，配置简单，但是一次只能一个客户端连接，适用于用来实验，不适用于生产环境。

> 解压hive安装包  bin/hive 启动即可使用

> 缺点：不同路径启动hive，每一个hive拥有一套自己的元数据，无法共享。

![1620824823305](./assets\1620824823305.png)



## 本地模式

> **本地模式**采用外部数据库来存储元数据，目前支持的数据库有：MySQL、Postgres、Oracle、MS SQL Server.在这里我们使用MySQL。

> 本地模式不需要单独起metastore服务，用的是跟hive在同一个进程里的metastore服务。也就是说当你启动一个hive 服务，里面默认会帮我们启动一个metastore服务。

> hive根据hive.metastore.uris 参数值来判断，如果为空，则为本地模式。

> 缺点是：每启动一次hive服务，都内置启动了一个metastore。

![1620824995824](./assets\1620824995824.png)

## **远程模式**

> **远程模式**下，需要单独起metastore服务，然后每个客户端都在配置文件里配置连接到该metastore服务。远程模式的metastore服务和hive运行在不同的进程里。
>
> 在生产环境中，建议用远程模式来配置Hive Metastore。
>
> 在这种情况下，其他依赖hive的软件都可以通过Metastore访问hive。

![1620825224479](./assets\1620825224479.png)

**远程模式下，需要配置hive.metastore.uris 参数来指定metastore服务运行的机器ip和端口，并且需要单独手动启动metastore服务。**

# **Hive的安装**

- 下载hive的安装包，这里我们选用hive的版本是2.1.1,软件包为：apache-hive-2.1.0-bin.tar.gz

- Hive下载地址:

~~~
http://archive.apache.org/dist/hive/
~~~

- 将apache-hive-2.1.0-bin.tar.gz上传到/export/software目录(node3)

- 解压Hive安装包并重命名 在node3上执行

  ```sh
  cd /export/software
  tar -zxvf apache-hive-2.1.0-bin.tar.gz  -C /export/server
  cd /export/server
  mv apache-hive-2.1.0-bin hive-2.1.0
  ```

- 修改hive的配置文件

  ~~~sh
  cd  /export/server/hive-2.1.0/conf
  cp hive-env.sh.template hive-env.sh
  vim hive-env.sh
  # 修改下面两个属性
  HADOOP_HOME=/export/server/hadoop-2.7.5 
  export HIVE_CONF_DIR=/export/server/hive-2.1.0/conf
  # 用 wq 保存退出
  
  ~~~

- 修改hive-site.xml

  ~~~sh
  cd  /export/server/hive-2.1.0/conf
  vim hive-site.xml
  ~~~

- 在该文件中添加以下内容

  ~~~xml
  <?xml version="1.0" encoding="UTF-8" standalone="no"?>
  <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
  <configuration>
  <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>123456</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://node3:3306/hive?createDatabaseIfNotExist=true&amp;useSSL=false</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>
    <property>
        <name>hive.metastore.schema.verification</name>
        <value>false</value>
    </property>
    <property>
      <name>datanucleus.schema.autoCreateAll</name>
      <value>true</value>
   </property>
   <property>
  	<name>hive.server2.thrift.bind.host</name>
  	<value>node3</value>
     </property>
  </configuration>
  ~~~

- 上传mysql的lib驱动包 /export/server/**hive-2.1.0**/lib目录下

- ##### 拷贝相关jar包

  ~~~sh
  # 将hive-2.1.0/jdbc/目录下的hive-jdbc-2.1.0-standalone.jar 拷贝到hive-2.1.0/lib/目录
  cp /export/server/hive-2.1.0/jdbc/hive-jdbc-2.1.0-standalone.jar /export/server/hive-2.1.0/lib/
  ~~~

- ##### 配置hive的环境变量

  ~~~sh
  vim /etc/profile
  
  export HIVE_HOME=/export/server/hive-2.1.0
  export PATH=:$HIVE_HOME/bin:$PATH
  # 运行以下命令，使profile生效。
  source /etc/profile
  
  ~~~

- 测试HIVE是否安装好

  - 启动HIVE

  ```sh
  # 第一种交互方式：
  hive
  
  # 出现下面结果就OK
  SLF4J: Class path contains multiple SLF4J bindings.
  SLF4J: Found binding in [jar:file:/export/server/hive-2.1.0/lib/hive-jdbc-2.1.0-standalone.jar!/org/slf4j/impl/StaticLoggerBinder.class]
  SLF4J: Found binding in [jar:file:/export/server/hive-2.1.0/lib/log4j-slf4j-impl-2.4.1.jar!/org/slf4j/impl/StaticLoggerBinder.class]
  SLF4J: Found binding in [jar:file:/export/server/hadoop-2.7.5/share/hadoop/common/lib/slf4j-log4j12-1.7.10.jar!/org/slf4j/impl/StaticLoggerBinder.class]
  SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
  SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]
  
  Logging initialized using configuration in jar:file:/export/server/hive-2.1.0/lib/hive-common-2.1.0.jar!/hive-log4j2.properties Async: true
  Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
  hive>
  ```

  - 创建一个表

  ~~~sql
  create database  mytest;
  
  # 出现下面结果就OK
  hive> create database  mytest;
  OK
  Time taken: 0.694 seconds
  ~~~

  - ##### 测试使用sql语句或者sql脚本进行交互

  ~~~sql
  hive -e "create database mytest2"
  
  # 出现下面结果就说明成功
  ve-log4j2.properties Async: true
  OK
  Time taken: 1.425 seconds
  ~~~



  ~~~sh
  mkdir /export/test
  cd /export/test
  
  vim  hive.sql
  
  create database mytest3;
  use mytest3;
  create table stu(id int,name string);
  
  #通过hive -f   来执行我们的sql脚本
  hive -f hive.sql
  
  # 如下则说明测试成功
  Logging initialized using configuration in jar:file:/export/server/hive-2.1.0/lib/hive-common-2.1.0.jar!/hive-log4j2.properties Async: true
  OK
  Time taken: 0.988 seconds
  OK
  Time taken: 0.014 seconds
  OK
  Time taken: 0.267 seconds
  ~~~

- 测试**第三种交互方式：Beeline Client**

   

  > hive经过发展，推出了第二代客户端beeline，但是beeline客户端不是直接访问metastore服务的，而是需要单独启动hiveserver2服务。
  >
  > 在hive运行的服务器上，首先**启动metastore服务，然后启动hiveserver2服务**。

  ~~~sh
  nohup /export/server/hive-2.1.0/bin/hive --service metastore &
  nohup /export/server/hive-2.1.0/bin/hive --service hiveserver2 &
  ~~~

  **nohup 和 & 表示后台启动**在node3上使用beeline客户端进行连接访问。

  ~~~sh
  beeline
  # 根据提醒进行以下操作:
  [root@node3 ~]# /export/server/hive-2.1.0/bin/beeline
  which: no hbase in (:/export/server/hive-2.1.0/bin::/export/server/hadoop-2.7.5/bin:/export/server/hadoop-2.7.5/sbin::/export/server/jdk1.8.0_241/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/export/server/mysql-5.7.29/bin:/root/bin)
  Beeline version 2.1.0 by Apache Hive
  
  
  beeline> !connect jdbc:hive2://node3:10000
  
  
  Connecting to jdbc:hive2://node3:10000
  Enter username for jdbc:hive2://node3:10000:root
  
  
  Enter password for jdbc:hive2://node3:10000:123456
  ~~~

**注意: 如果报出以下, 请修改 hadoop中 core-site.xml文件**

**错误信息为:   User: root is not allowed to impersonate root**

- 解决方案: 在node1的 hadoop的 core-site.xml文件中添加一下内容:

  ~~~xml
  <property> 
  <name>hadoop.proxyuser.root.hosts</name>
  <value>*</value> 
  </property> 
  <property> 
  <name>hadoop.proxyuser.root.groups</name> 
  <value>*</value> 
  </property>
  ~~~

  ```sh
  cd /export/server/hadoop-2.7.5/etc/hadoop
  scp core-site.xml node2:$PWD
  scp core-site.xml node3:$PWD
  # 操作完后重复上一步操作
  # 下面这个表示成功
  21/05/12 22:36:21 [main]: WARN jdbc.HiveConnection: Request to set autoCommit to false; Hive does not support autoCommit=false.
  Transaction isolation: TRANSACTION_REPEATABLE_READ
  0: jdbc:hive2://node3:10000> 
  ```


### Hive一键启动脚本

> 这里，我们写一个expect脚本，可以一键启动beenline，并登录到hive。expect是建立在tcl基础上的一个自动化交互套件, 在一些需要交互输入指令的场景下, 可通过脚本设置自动进行交互通信。

1. 安装expect

   ~~~sh
   yum install expect
   ~~~

2. 创建启动脚本

   ```sh
   cd /export/server/hive-2.1.0/bin
   vim  beenline.exp
   
   #!/bin/expect
   spawn beeline
   set timeout 5
   expect "beeline>"
   send "!connect jdbc:hive2://node3:10000\r"
   expect "Enter username for jdbc:hive2://node3:10000:"
   send "root\r"
   expect "Enter password for jdbc:hive2://node3:10000:"
   send "123456\r"
   interact
   
   
   chmod 777 beenline.exp
   ```

3. 启动beeline

   ~~~sh
   expect beenline.exp
   ~~~
