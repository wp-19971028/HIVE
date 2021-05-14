# Hive 命令行

语法结构

~~~sql
hive [-hiveconf x=y]* [<-i filename>]* [<-f filename>|<-e query-string>] [-S]
说明：
1、-i  从文件初始化HQL。
2、-e 从命令行执行指定的HQL 
3、-f  执行HQL脚本 
4、-v  输出执行的HQL语句到控制台 
5、-p <port> connect to Hive Server on port number 
6、-hiveconf x=y Use this to set hive/hadoop configuration variables.  设置hive运行时候的参数配置
~~~

# Hive参数配置方式

开发Hive应用时，不可避免地需要设定Hive的参数。设定Hive的参数可以调优HQL代码的执行效率，或帮助定位问题。然而实践中经常遇到的一个问题是，为什么设定的参数没有起作用？这通常是错误的设定方式导致的。

 

**对于一般参数，有以下三种设定方式：**

- 配置文件 

- 命令行参数 

- 参数声明 

 

## 配置文件 

> Hive的配置文件包括
>
> - 用户自定义配置文件：$HIVE_CONF_DIR/hive-site.xml 
>
> - 默认配置文件：$HIVE_CONF_DIR/hive-default.xml 
>
> 用户自定义配置会覆盖默认配置。
>
> 另外，Hive也会读入Hadoop的配置，因为Hive是作为Hadoop的客户端启动的，Hive的配置会覆盖Hadoop的配置。
>
> 配置文件的设定对本机启动的所有Hive进程都有效。

## 命令行参数 

> 启动Hive（客户端或Server方式）时，可以在命令行添加-hiveconf param=value来设定参数，例如：
>
> ```sh 
> hive -hiveconf hive.root.logger=INFO,console
> ```
>
> 这一设定对本次启动的Session（对于Server方式启动，则是所有请求的Sessions）有效。

## 参数声明 

> 可以在HQL中使用SET关键字设定参数，例如：
>
> ```sql
> set mapred.reduce.tasks=100;
> ```
>
> 这一设定的作用域也是session级的。
>
> 上述三种设定方式的优先级依次递增。即参数声明覆盖命令行参数，命令行参数覆盖配置文件设定。注意某些系统级的参数，例如log4j相关的设定，必须用前两种方式设定，因为那些参数的读取在Session建立以前已经完成了。

# 参数的优先级

参数声明 > 命令行参数> 配置文件