# SELECT语句

## 语句结构

~~~sql
SELECT [ALL | DISTINCT]select_expr, select_expr, ...
FROM table_reference
[WHERE where_condition]
[GROUP BYcol_list]
[HAVING where_condition]
[ORDER BYcol_list]
[CLUSTER BYcol_list
  | [DISTRIBUTE BY col_list] [SORT BY col_list]
]
[LIMIT number]
~~~

**解释:**

1. **ORDER BY**用于全局排序，就是对指定的所有排序键进行全局排序，使用ORDER BY的查询语句，最后会用一个Reduce Task来完成全局排序。

2. **sort by**用于分区内排序，即每个Reduce任务内排序**。**则sort by只保证每个reducer的输出有序，不保证全局有序。

3. **distribute by(**字段)根据指定的字段将数据分到不同的reducer，且分发算法是hash散列。

4. **cluster by**(字段) 除了具有Distribute by的功能外，还兼具sort by的排序功能。因此，如果分桶和sort字段是同一个时，此时，cluster by = distribute by + sort by

## 全表查询

~~~sql
SELECT  *
FROM    score;
~~~

## 查询特定的字段

```sql
SELECT  s_id,c_id
FROM    score;
```

## 起别名

```sql
SELECT  s_id as myid ,c_id
FROM    score;
```

## 常用函数

~~~sql
-- 求总行数 (count)
SELECT  count(1)
FROM    score;
-- 求分数的最大值（max）
SELECT  max(s_score)
FROM    score;
-- 求分数的最小值（min）
SELECT  min(s_score)
FROM    score;
-- 求分数的总和（sum）
SELECT  sum(s_score)
FROM    score;
-- 求分数的平均值（avg）
SELECT  avg(s_score)
FROM    score;

~~~

## LIMIT语句

```sql
-- 查前三
select * from score limit 3;

-- 第一个是测试数据 查除第一个往后的前三
select * from score limit 1,3;

```



## **WHERE语句**

1. 使用WHERE 子句，将不满足条件的行过滤掉。

2. WHERE 子句紧随 FROM 子句。

3. 案例实操

~~~sql
-- 查询出分数大于60的数据
select * from score where s_score > 60;
~~~



# 运算符

| **操作符**              | **支持的数据类型** | **描述**                                                     |
| ----------------------- | ------------------ | ------------------------------------------------------------ |
| A=B                     | 基本数据类型       | 如果A等于B则返回TRUE，反之返回FALSE                          |
| A<=>B                   | 基本数据类型       | 如果A和B都为NULL，则返回TRUE，其他的和等号（=）操作符的结果一致，如果任一为NULL则结果为NULL |
| A<>B, A!=B              | 基本数据类型       | A或者B为NULL则返回NULL；如果A不等于B，则返回TRUE，反之返回FALSE |
| A<B                     | 基本数据类型       | A或者B为NULL，则返回NULL；如果A小于B，则返回TRUE，反之返回FALSE |
| A<=B                    | 基本数据类型       | A或者B为NULL，则返回NULL；如果A小于等于B，则返回TRUE，反之返回FALSE |
| A>B                     | 基本数据类型       | A或者B为NULL，则返回NULL；如果A大于B，则返回TRUE，反之返回FALSE |
| A>=B                    | 基本数据类型       | A或者B为NULL，则返回NULL；如果A大于等于B，则返回TRUE，反之返回FALSE |
| A [NOT] BETWEEN B AND C | 基本数据类型       | 如果A，B或者C任一为NULL，则结果为NULL。如果A的值大于等于B而且小于或等于C，则结果为TRUE，反之为FALSE。如果使用NOT关键字则可达到相反的效果。 |
| A IS NULL               | 所有数据类型       | 如果A等于NULL，则返回TRUE，反之返回FALSE                     |
| A IS NOT NULL           | 所有数据类型       | 如果A不等于NULL，则返回TRUE，反之返回FALSE                   |
| IN(数值1, 数值2)        | 所有数据类型       | 使用 IN运算显示列表中的值                                    |
| A [NOT] LIKE B          | STRING 类型        | B是一个SQL下的简单正则表达式，如果A与其匹配的话，则返回TRUE；反之返回FALSE。B的表达式说明如下：‘x%’表示A必须以字母‘x’开头，‘%x’表示A必须以字母’x’结尾，而‘%x%’表示A包含有字母’x’,可以位于开头，结尾或者字符串中间。如果使用NOT关键字则可达到相反的效果。 |
| A RLIKE B, A REGEXP B   | STRING 类型        | B是一个正则表达式，如果A与其匹配，则返回TRUE；反之返回FALSE。匹配使用的是JDK中的正则表达式接口实现的，因为正则也依据其中的规则。例如，正则表达式必须和整个字符串A相匹配，而不是只需与其字符串匹配。 |

## 条件查询

**案例实操**

```sql
--（1）查询分数等于80的所有的数据
select * from score where s_score = 80;
--（2）查询分数在80到100的所有数据
select * from score where s_score between 80 and 100;
--（3）查询成绩为空的所有数据
select * from score where s_score is null;
--（4）查询成绩是80和90的数据
select * from score where s_score in(80,90);
```

## 模糊查询LIKE和RLIKE

> (1）使用LIKE运算选择类似的值
>
> (2）选择条件可以包含字符或数字:
>
> % 代表零个或多个字符(任意个字符)。
>
> _ 代表一个字符。
>
> 3）RLIKE子句是Hive中这个功能的一个扩展，其可以通过Java的正则表达式这个更强大的语言来指定匹配条件。

**案例实操**

```sql
（1）查找以8开头的所有成绩
select * from score where s_score like '8%';
（2）查找第二个数值为9的所有成绩数据
select * from score where s_score like '_9%';
（3）查找成绩中含9的所有成绩数据
select * from score where s_score rlike '[9]';		
```

## 逻辑运算符

| 操作符 | 含义   |
| ------ | ------ |
| AND    | 逻辑并 |
| OR     | 逻辑或 |
| NOT    | 逻辑否 |

案例实操

```sql
--（1）查询成绩大于80，并且s_id是01的数据
select * from score where s_score >80 and s_id = '01';
--（2）查询成绩大于80，或者s_id  是01的数
select * from score where s_score > 80 or s_id = '01';
--（3）查询s_id  不是 01和02的学生
select * from score where s_id not in ('01','02');
```





# 分组

> ##### **GROUP BY语句**
>
> GROUP BY语句通常会和聚合函数一起使用，按照一个或者多个列队结果进行分组，然后对每个组执行聚合操作。注意使用group  by分组之后，select后面的字段只能是分组字段和聚合函数。

案例实操：

~~~sql
--（1）计算每个学生的平均分数
select s_id ,avg(s_score) from score group by s_id;

--（2）计算每个学生最高成绩
select s_id ,max(s_score) from score group by s_id;
~~~



## HAVING语句

> **having与where不同点**
>
> （1）where针对表中的列发挥作用，查询数据；having针对查询结果中的列发挥作用，筛选数据。
>
> （2）where后面不能写分组函数，而having后面可以使用分组函数。
>
> （3）having只用于group by分组统计语句。

**案例实操：**

~~~sql
-- 求每个学生的平均分数
select s_id ,avg(s_score) from score group by s_id;
-- 求每个学生平均分数大于85的人
select s_id ,avg(s_score) avgscore from score group by s_id having avgscore > 85;
~~~



# JOIN语句

## 右外连接（RIGHT OUTER JOIN）

> 右外连接：JOIN操作符右边表中符合WHERE子句的所有记录将会被返回。

~~~sql
select * from teacher t right join course c on t.t_id = c.t_id;
~~~

##  内连接（INNER JOIN)

内连接：只有进行连接的两个表中都存在与连接条件相匹配的数据才会被保留下来。

```sql
select * from teacher t inner join course c on t.t_id = c.t_id;
```

##  左外连接（LEFT OUTER JOIN)

> 左外连接：JOIN操作符左边表中符合WHERE子句的所有记录将会被返回。
>
> 查询老师对应的课程

~~~sql
select * from teacher t left join course c on t.t_id = c.t_id;
~~~

##  满外连接（FULL OUTER JOIN)

> 满外连接：将会返回所有表中符合WHERE语句条件的所有记录。如果任一表的指定字段没有符合条件的值的话，那么就使用NULL值替代。

~~~sql
SELECT * FROM teacher t FULL JOIN course c ON t.t_id = c.t_id ;
~~~

## 多表连接

> 注意：连接 n个表，至少需要n-1个连接条件。例如：连接三个表，至少需要两个连接条件。
>
> 多表连接查询，查询老师对应的课程，以及对应的分数，对应的学生

~~~sql
select * from teacher t 
left join course c 
on t.t_id = c.t_id
left join score s 
on s.c_id = c.c_id
left join student stu 
on s.s_id = stu.s_id;

~~~

> 大多数情况下，Hive会对每对JOIN连接对象启动一个MapReduce任务。本例中会首先启动一个MapReduce job对表teacher和表course进行连接操作，然后会再启动一个MapReduce job将第一个MapReduce job的输出和表score;进行连接操作。

# 排序

## OrderBy全局排序

> Order By：全局排序，一个reduce
>
> 1、使用 ORDER BY 子句排序
>
> ASC（ascend）: 升序（默认）
>
> DESC（descend）: 降序
>
> 2、ORDER BY 子句在SELECT语句的结尾。

案例实操

```sql
--（1）查询学生的成绩，并按照分数降序排列
SELECT * FROM student s LEFT JOIN score sco ON s.s_id = sco.s_id ORDER BY sco.s_score DESC;
-- (2) 按照分数的平均值排序
select s_id ,avg(s_score) avg from score group by s_id order by avg;
-- (3)按照学生id和平均成绩进行排序
select s_id ,avg(s_score) avg from score group by s_id order by s_id,avg;
```

##  **Sort** By每个MapReduce内部局部排序

Sort By：每个MapReduce内部进行排序，对全局结果集来说不是排序。

~~~sql
-- 1)设置reduce个数
 set mapreduce.job.reduces=3;
-- 2)查看设置reduce个数
 set mapreduce.job.reduces;
-- 3）查询成绩按照成绩降序排列
 select * from score sort by s_score;
-- 4)将查询结果导入到文件中（按照成绩降序排列）
 insert overwrite local directory '/export/server/hivedatas/sort' select * from score sort by s_score;
~~~

## Distribute By-分区排序

> Distribute By：类似MR中partition，进行分区，结合sort by使用。
>
> ​	注意，Hive要求DISTRIBUTE BY语句要写在SORT BY语句之前。
>
> 对于distribute by进行测试，一定要分配多reduce进行处理，否则无法看到distribute by的效果。
>
>

**案例实操：**

先按照学生id进行分**区**，再按照学生成绩进行排序。

~~~sql
1)设置reduce的个数，将我们对应的s_id划分到对应的reduce当中去
set mapreduce.job.reduces=7;
2)通过distribute by进行数据的分区
insert overwrite local directory '/export/server/hivedatas/sort' select * from score distribute by s_id sort by s_score;
~~~

## Cluster By

> 当distribute by和sort by字段相同时，可以使用cluster by方式。
>
> cluster by除了具有distribute by的功能外还兼具sort by的功能。但是排序只能是倒序排序，不能指定排序规则为ASC或者DESC。
>
> 以下两种写法等价:

```sql
select * from score cluster by s_id; 
select * from score distribute by s_id sort by s_id;
```

