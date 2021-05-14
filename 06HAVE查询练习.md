# 准备数据

~~~sql
### 数据的准备
# 创建一个数据库
create database if not exists day13_hive;
# 使用这个数据库
use day13_hive;
# 创建对应的表 ：学生表和成绩表
create external table student (s_id string,s_name string,s_birth string , s_sex string ) row format delimited fields terminated by '\t';
create external table score (s_id string,c_id string,s_score int) row format delimited fields terminated by '\t';
create external table teacher (t_id string,t_name string) row format delimited fields terminated by '\t';
# 上传数据到hdfs指定的目录 /hivedatas
hdfs dfs -put /export/server/hivedatas/score.csv /hivedatas
hdfs dfs -put /export/server/hivedatas/student.csv /hivedatas
hdfs dfs -put /export/server/hivedatas/teacher.csv /hivedatas
# 将数据导入到学生表和成绩表中
load data inpath '/hivedatas/score.csv' into table score;
load data inpath '/hivedatas/student.csv' into table student;
load data inpath '/hivedatas/teacher.csv' into table teacher;
~~~

# 数据操作

~~~sql
### 数据的操作
# 将输出的reduce个数调为1
set mapred.reduce.tasks=1;
# 查询"01"课程比"02"课程成绩高的学生的信息及课程分数
select * from 
(select * from
(select s_id as 01_sid,s_score as 01_score from score where c_id='01') as tmp1
join
(select s_id as 02_sid,s_score as 02_score from score where c_id='02') as tmp2
on tmp1.01_sid=tmp2.02_sid where tmp1.01_score >tmp2.02_score) score
join student s on score.01_sid = s.s_id;
-- 2、查询"01"课程比"02"课程成绩低的学生的信息及课程分数
SELECT a.* ,b.s_score AS 01_score,c.s_score AS 02_score FROM 
    student a LEFT JOIN score b ON a.s_id=b.s_id AND b.c_id='01' 
     JOIN score c ON a.s_id=c.s_id AND c.c_id='02' WHERE b.s_score<c.s_score;


-- 3、查询平均成绩大于等于60分的同学的学生编号和学生姓名和平均成绩
SELECT b.s_id,b.s_name,ROUND(AVG(a.s_score),2) AS avg_score FROM 
    student b 
    JOIN score a ON b.s_id = a.s_id
    GROUP BY b.s_id,b.s_name HAVING ROUND(AVG(a.s_score),2)>=60;

-- 4、查询平均成绩小于60分的同学的学生编号和学生姓名和平均成绩(包括有成绩的和无成绩的)
SELECT b.s_id,b.s_name,ROUND(AVG(a.s_score),2) AS avg_score FROM 
    student b 
    LEFT JOIN score a ON b.s_id = a.s_id
    GROUP BY b.s_id,b.s_name HAVING ROUND(AVG(a.s_score),2)<60
    UNION ALL
SELECT a.s_id,a.s_name,0 AS avg_score FROM 
    student a 
    WHERE a.s_id NOT IN (
                SELECT DISTINCT s_id FROM score);

-- 5、查询所有同学的学生编号、学生姓名、选课总数、所有课程的总成绩
SELECT a.s_id,a.s_name,COUNT(b.c_id) AS sum_course,SUM(b.s_score) AS sum_score FROM 
    student a 
    LEFT JOIN score b ON a.s_id=b.s_id
    GROUP BY a.s_id,a.s_name;

-- 6、查询"李"姓老师的数量
select count(t_id) from teacher where t_name like '李%';

-- 7、查询学过"张三"老师授课的同学的信息
SELECT a.*
FROM student a LEFT JOIN score b ON a.s_id = b.s_id WHERE b.c_id  IN  (
SELECT c.c_id
FROM course c LEFT JOIN teacher t ON c.t_id = t.t_id WHERE t.t_name = '张三'
) ;

-- 8、查询没学过"张三"老师授课的同学的信息
SELECT s.*
FROM student s LEFT JOIN (
SELECT a.s_id
FROM student a LEFT JOIN score b ON a.s_id = b.s_id WHERE b.c_id  IN  (
SELECT c.c_id
FROM course c LEFT JOIN teacher t ON c.t_id = t.t_id WHERE t.t_name = '张三'
) 
) ss ON s.s_id = ss.s_id WHERE ss.s_id IS NULL;

-- 9、查询学过编号为"01"并且也学过编号为"02"的课程的同学的信息

select a.* from 
    student a,score b,score c 
where a.s_id = b.s_id  and a.s_id = c.s_id and b.c_id='01' and c.c_id='02';

-- 10、查询学过编号为"01"但是没有学过编号为"02"的课程的同学的信息
SELECT qq.*
FROM (
SELECT s.* 
FROM student s LEFT JOIN score sco ON s.s_id = sco.s_id LEFT JOIN course c ON sco.c_id = c.c_id WHERE c.c_id='01'  
) qq
LEFT JOIN (
SELECT stu.* 
FROM student stu LEFT JOIN score mysco ON stu.s_id = mysco.s_id LEFT JOIN course cou ON mysco.c_id = cou.c_id WHERE cou.c_id='02'  
) pp
ON qq.s_id = pp.s_id WHERE pp.s_id IS NULL;

-- 11、查询没有学全所有课程的同学的信息

SELECT ss.s_id
FROM (
SELECT stu.s_id,COUNT(stu.s_id) AS num
FROM student stu LEFT JOIN score sco ON stu.s_id = sco.s_id LEFT JOIN course cou ON sco.c_id = cou.c_id 
GROUP BY stu.s_id
) ss WHERE ss.num < 3

-- 12、查询至少有一门课与学号为"01"的同学所学相同的同学的信息
SELECT stu.*
FROM student stu LEFT JOIN 
(
SELECT s.s_id
FROM score s WHERE s.c_id IN(
SELECT  c_id FROM score WHERE s_id = '01'
)GROUP BY s_id
) pp ON stu.s_id = pp.s_id WHERE pp.s_id IS NOT  NULL;
~~~



