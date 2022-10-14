
# 1 产生背景

- https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/cdh_components.html

* MapReduce编程不便
* 传统RDBMS人员需要
* FaceBook开源，用于解决海量结构化日志的数据统计问题
* 构建在Hadoop之上的数据仓库(不准确)
  * 数据存放在distributed storage上（Hadoop）
  * 元数据是存储在metastore对应的底层数据库中（MySQL）
* Hive提供SQL查询语句：HQL
* 底层支持多种不同的引擎
  * MR
  * Tez
  * Spark==> 2.x默认



# 2 Hive体系架构

- SQL Parser：将SQL字符串解析为AST
- Physical Plan：将AST编译生成逻辑执行计划
- Query Optimizer：对逻辑执行计划进行优化
- Execution：将逻辑执行计划转化为可运行的物理计划（MR/Spark）

<img src="pictures/Hive/Hive架构.png" style="zoom:50%;" />

* client：shell、thrift（协议，把Hive变成服务）/jdbc(server/jdbc)、WebUI（HUE/Zeppelin）

- Hive vs RDBMS

| 对比项               | Hive       | RDBMS    |
| -------------------- | ---------- | -------- |
| 查询语言             | HQL        | SQL      |
| 数据存储             | 分布式HDFS | 单机     |
| 执行器               | MR         | executor |
| 处理数据规模         | 大         | 小       |
| 延迟                 | 高         | 低       |
| 事务                 | 支持但无用 | 支持     |
| insert/update/delete | 支持       | 支持     |

- 适用场景
  - 批处理/离线处理（用于分析）
  - 延时性高
  - 尽量少涉及update或者delete操作，虽然支持
- Hive优缺点
  - 优点：易上手、比MR用起来简单
  - 简单易上手
  - 为超大数据集设计的计算
  
  - 统一元数据管理
  
    * Hive数据是存放在HDFS上
    * 元数据信息（描述数据的数据）是存放在metastore所在的MySQL（实践中）
    * SQL on Hadoop：Hive、Spark、SQL、impala........（方便移植,通过一个参数切换）
  
  - Hive是一个客户端（提交机器），没有集群概念（只需要在需要提交的机器上安装就可以）
  
    * 职责：将SQL翻译成底层对应的执行引擎作业
  - 缺点：高延时
- derby问题
  - 单session，多人使用在同一个目录中会有metastore冲突

## metastore

**1 存储Hive版本的元数据表** 

- VERSION

**2 Hive数据库相关的元数据表**

- DBS
- DATABASE_PARAMS

**3 Hive表和视图相关的元数据表**

- 主要有TBLS、TABLE_PARAMS、TBL_PRIVS，这三张表通过TBL_ID关联。
- TBLS 该表中存储Hive表、视图、索引表的基本信息。
- TABLE_PARAMS 该表存储表/视图的属性信息。
- TBL_PRIVS 该表存储表/视图的授权信息

**4 Hive文件存储信息相关的元数据表**

- 主要涉及SDS、SD_PARAMS、SERDES、SERDE_PARAMS
- 由于HDFS支持的文件格式很多，而建Hive表时候也可以指定各种文件格式，Hive在将HQL解析成MapReduce时候，需要知道去哪里，使用哪种格式去读写HDFS文件，而这些信息就保存在这几张表中。

  **table**

- SDS 该表保存文件存储的基本信息，如INPUT_FORMAT、OUTPUT_FORMAT、是否压缩等。
- TBLS 表中的SD_ID与该表关联，可以获取Hive表的存储信息。
- SD_PARAMS 该表存储Hive存储的属性信息，在创建表时候使用
- SERDES 该表存储序列化使用的类信息
- SERDE_PARAMS 该表存储序列化的一些属性、格式信息,比如：行、列分隔符

**5 Hive表字段相关的元数据表**

- COLUMNS_V2 该表存储表对应的字段信息

**6 Hive表分区相关的元数据表**

- PARTITIONS 该表存储表分区的基本信息
- PARTITION_KEYS 该表存储分区的字段信息
- PARTITION_KEY_VALS 该表存储分区字段值
- PARTITION_PARAMS 该表存储分区的属性信息



# 3 部署

版本 apache hive2.3.8

- 目录结构
  - auxlib   UDF函数编程(CDH版本有)
  - conf      Hive配置
  - lib         jar包
  - bin       脚本

## 配置文件

```shell
# step 1
# 解压
tar -zxvf hive-1.1.0-cdh5.15.1.tar.gz -C ~/app/

# step 2
# 写入环境变量
vi ~/.bash_profle
export HIVE_HOME=/home/hadoop/app/hive-1.1.0-cdh5.15.1
export PATH=$HIVE_HOME/bin:$PATH

# step 3
# 修改配置 hive-env.sh
HADOOP_HOME=/home/hadoop/app/hadoop-2.6.0-cdh5.15.1  # 增加

# 在lib目录下加MySQL的jar包
## mysql-connector-java-5.1.27-bin.jar
```

* [hive-site.xml](https://github.com/apache/hive/blob/master/data/conf/hive-site.xml)

```shell
# 自己配置，如没有
cd /home/hadoop/app/hive-1.1.0-cdh5.15.1/conf

# hive-log4j2.properties 日志配置
```

 hive-site.xml

```xml
<property>
  <name>javax.jdo.option.ConnectionURL</name>
  <value>jdbc:mysql://hadoop000:3306/hive2?createDatabaseIfNotExist=true&amp;useUnicode=true&amp;characterEncoding=UTF-8</value>
</property>

<property>
  <name>javax.jdo.option.ConnectionDriverName</name>
  <value>com.mysql.jdbc.Driver</value>
</property>

<property>
  <name>javax.jdo.option.ConnectionUserName</name>
  <value>root</value>
</property>

<property>
  <name>javax.jdo.option.ConnectionPassword</name>
  <value>xxx@123</value>
</property>

<property>
  <name>hive.cli.print.header</name>
  <value>true</value>
</property>

<property>
  <name>hive.cli.print.current.db</name>
  <value>true</value>
</property>

<property>
  <name>hive.server2.thrift.port</name>
  <value>10000</value>
</property>

<property>
    <name>hive.metastore.schema.verification</name>
    <value>false</value>
</property>

<property>
  <name>hive.server2.webui.host</name>
  <value>hadoop000</value>
</property>
```

- 第一次可能需要初始化

```shell
./schematool -initSchema -dbType mysql
./hive

# 其他设置
add file file:///path/to/file;
list files;

list jars;
add jar file:///path/to/jar;
delete jar file:///path/to/jar;
```

- metastore部署

```shell
hive --service metastore &
```

> 默认的default数据库存放路径(hdfs)
>
> /user/hive/warehouse

## beeline

- hive-site.xml

```xml
<property>
	<name>hive.server2.webui.host</name>
  <value>hadoop000</value>
</property>
<property>
	<name>hive.server2.webui.port</name>
  <value>19990</value>
</property>
```

- core-site.xml(hadoop)

```xml
<property>
    <name>hadoop.proxyuser.hadoop.hosts</name>
    <value>*</value>
</property>
<property>
    <name>hadoop.proxyuser.hadoop.groups</name>
    <value>*</value>
</property>
```

- hiveServer2

```shell
nohup sh hiveserver2 &
```

- beeline

```shell
# 在修改core-site.xml(hadoop)后 刷新权限
hadoop dfsadmin -refreshSuperUserGroupsConfiguration
# hive.server2.thrift.port 默认端口 10000
$HIVE_HOME/bin/beeline -u jdbc:hive2://hadoop000:10000/default -n hadoop
```

## JDBC

```java
public class JDBCAPI {

    private static String driverName = "org.apache.hive.jdbc.HiveDriver";

    public static void main(String[] args) throws SQLException {
        try {
            Class.forName(driverName);
        } catch (ClassNotFoundException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
            System.exit(1);
        }

        Connection con = DriverManager.getConnection("jdbc:hive2://hadoop000:10000/default", "hadoop", "");
        Statement stmt = con.createStatement();

        // select * query
        String sql = "select * from emp";
        System.out.println("Running: " + sql);
        ResultSet res = stmt.executeQuery(sql);
        while (res.next()) {
            System.out.println(String.valueOf(res.getInt("empno")) + "\t" + res.getString("ename"));
        }
    }
}
```

- pom.xml

```xml
<dependency>
    <groupId>org.apache.hive</groupId>
    <artifactId>hive-jdbc</artifactId>
    <version>1.1.0</version>
</dependency>
```

## 参数设置

* 参数设置几种方法
  * hive-site.xml  全局
    		场景A：SQL1   set hive.smalltable.filesize=30M
      		场景B：SQL2   set hive.smalltable.filesize=50M
  * 优先级
    - hive-site.xml < hive --hiveconf (启动时加载)< set a=b
* MySQL驱动
  * 拷贝驱动到$ HIVE_HOME/lib 
  * 前提有安装MySQL数据库(不要8.x)
  * https://www.cnblogs.com/julyme/p/5969626.html
  * Hadoop 中默认default路径
    * /user/hive/warehouse
    * 每张表存放在对应数据库中，没指定就是默认
    * 自己创建/user/hive/warehouse/database_name.db



# 4 HiveSQL

##  4.1 参数设置

- [参数文档](https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties#ConfigurationProperties-hive.mapred.mode)（如 hive.execution.engine）

- 常用分隔符  

  ```shell
  \t 空格 \u007
  ```

### 设置方式

1.  hive-site.xml  全局
2.  命令行 hive --hiveconf
3.  命令窗口set方式 局部

优先级：方式3 > 方式2 > 方式1

### hive命令

```shell
hive -e          # SQL语句
hive -f          # SQL文件
hive --hiveconf  # 设置参数
hive -i          # 定义UDF函数

dfs -ls /user/hive/warehouse;  # 命令窗口内执行查看dfs存储
```

```sql
set hive.execution.engine;     # 查询 hive.execution.engine
set hive.execution.engine=mr;  # 赋值
set hive.fetch.task.conversion; # 确定哪些逻辑跑mr
set hive.exec.mode.local.auto=true; # 本地模式
set hive.mapred.mode;  # strict / nonstrict

# 控制map数量
set hive.exec.parallel;  # 是否开启并行计算（互不依赖job 前提资源足够）
set hive.exec.parallel.thread.number; # 并行个数
set hive.auto.convert.join;  # converting common join into mapjoin based on the input file size
set mapreduce.input.fileinputforrmat.split.maxsize;
set mapreduce.job.jvm.numtasks;  # jvm复用

# 控制reduce数量-->输出文件个数
# hive 源码 mapRedTask; Utilities.estimateNumberOfReducers
set mapreduce.job.reduces;  # 有3个参数 综合控制
```

hive源码

```java
// 控制reduce数量
public static int estimateReducers(long totalInputFileSize, long bytesPerReducer,
    int maxReducers, boolean powersOfTwo) {
    double bytes = Math.max(totalInputFileSize, bytesPerReducer);
    int reducers = (int) Math.ceil(bytes / bytesPerReducer);
    reducers = Math.max(1, reducers);
    reducers = Math.min(maxReducers, reducers);

    int reducersLog = (int)(Math.log(reducers) / Math.log(2)) + 1;
    int reducersPowerTwo = (int)Math.pow(2, reducersLog);

    if (powersOfTwo) {
      // If the original number of reducers was a power of two, use that
      if (reducersPowerTwo / 2 == reducers) {
        // nothing to do
      } else if (reducersPowerTwo > maxReducers) {
        // If the next power of two greater than the original number of reducers is greater
        // than the max number of reducers, use the preceding power of two, which is strictly
        // less than the original number of reducers and hence the max
        reducers = reducersPowerTwo / 2;
      } else {
        // Otherwise use the smallest power of two greater than the original number of reducers
        reducers = reducersPowerTwo;
      }
    }
    return reducers;
  }
```



## 4.2 表操作

<img src="pictures/Hive/表存储结构.png" style="zoom:50%;" />

- database   HDFS一个目录
- table   HDFS一个目录
- partition  分区表   HDFS一个目录
  - data    文件
  - bucket 分桶   HDFS一个文件

### 4.2.0 数据类型

int/bigint/double/float/double/string/decimal

### 4.2.1 [DDL(Hive Data Definition Language)](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL)

#### database

```sql
show database like '*';        # 查询数据库信息
!clear;                        # 清空屏幕

create database xxx;                                      # 存储在 /user/hive/warehouse/xxx.db
create database xxx location '/user/hive/warehouse/xxx';  # 存储在 /user/hive/warehouse/xxx
create database xxx with DBPROPERTIES('creator'='andy','date'='2021-09-01');
desc database extended xxx;
```

#### table

```sql
desc extended table_name;
desc formatted table_name;
```

#### alter

```sql
-- 增加1列
ALTER TABLE dept ADD COLUMNS(info string);

-- 修改列
ALTER TABLE dept CHANGE info infos string;

-- 替换现有所有列
ALTER TABLE dept REPLACE COLUMNS(deptno int, dname string, loc string);
```

#### create

- 建表语句解析

```sql
CREATE TABLE：       # 指定要创建的表的名字
col_name data_type： # 列名以及对应的数据类型，多个列之间使用逗号分隔
PARTITIONED BY：     # 指定分区
CLUSTERED BY：       # 排序、分桶
ROW FORMAT：         # 指定数据的分隔符等信息
STORED AS：          # 指定表存储的数据格式：textfile rc orc parquet
LOCATION：           # 指定表在文件系统上的存储路径
AS select_statement: # 通过select的sql语句的结果来创建表
```

- 创建结果表案例

```sql
CREATE TABLE emp(
empno int, 
ename string, 
job string,
mgr string, 
hiredate string, 
sal double, 
comm double, 
deptno int
) ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t';
```

- 创建中间临时表

```sql
-- 只拷贝表结构
create table emp2 like emp;
-- 拷贝数据和表结构
create table emp3 as select * from emp;
```



###  4.2.2 DML

#### load

- 加载数据解析

```sql
LOAD DATA        # 加载数据
LOCAL            # 有：从本地[Hive客户端加载数据到Hive表; 无：从文件系统加载数据到Hive表
INPATH           # 加载数据的路径
OVERWRITE        # 有：覆盖已有的数据  overwrite; 无：追加 append
INTO TABLE       # 加载数据到哪个表中
```

- 加载数据

```sql
LOAD DATA [LOCAL] INPATH 'filepath' [OVERWRITE] INTO TABLE tablename [PARTITION (partcol1=val1, partcol2=val2 …)]

-- local
load data local inpath '/home/hadoop/data/emp.txt' overwrite into table emp;

-- hdfs
load data inpath '/data/hive/emp.txt' into table emp;
```

#### insert

```sql
INSERT OVERWRITE TABLE tablename1 select_statement1 FROM from_statement;
INSERT INTO TABLE tablename1 select_statement1 FROM from_statement;

-- 写法2
FROM from_statement
INSERT OVERWRITE TABLE tablename1 select_statement1
-- 案例1 一份数据 多份拷贝
FROM emp
INSERT OVERWRITE TABLE result1 A业务 
INSERT OVERWRITE TABLE result2 B业务 
-- 案例2 不使用*
-- 此时需要字段（数量，类型）对齐
INSERT OVERWRITE TABLE emp2 
select empno,job ,ename,mgr,hiredate,sal ,comm,deptno from emp;

-- 一次插入多条数据（不建议使用）
-- 每次操作都会生成小文件
INSERT INTO TABLE dept 
VALUES (95271, 'DEV', 'BJ'),(95281, 'QA', 'SZ');
```

- 导入分布式文件系统（hdfs）

```sql
INSERT OVERWRITE DIRECTORY '/home/hadoop/hivetmp'
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
SELECT * FROM emp;
```

- 导入本地文件系统

```sql
INSERT OVERWRITE LOCAL DIRECTORY '/home/hadoop/hivetmp'
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
SELECT * FROM emp;
```

- 集群间数据迁移

```sql
EXPORT TABLE emp to '/hive_export/emp';  # 导到hdfs

-- 会根据metadata自动创建表emp_import
IMPORT TABLE emp_import
FROM '/hive_export/emp';
```

#### truncate/drop

```sql
# 只删除数据
# 外部表无效
truncate table xxx;

# 删除元数据和数据
drop table xxx;

# 删除分区数据
alter table xxx drop partition (l_date='20211010');
```

#### select

```sql
-- 查看执行计划
explain extended select name, count(1) from login group by name;
```

- hive 0.10.0为了执行效率考虑，简单的查询，就是只是select，不带count,sum,group by这样的，都不走map/reduce，直接读取hdfs文件进行filter过滤
- 这样做的好处就是不新开mr任务，执行效率要提高不少，但是不好的地方就是用户界面不友好，有时候数据量大还是要等很长时间，但是又没有任何返回

```sql
# 这个参数设置为more，简单查询就不走map/reduce了，设置为minimal，就任何简单select都会走map/reduce
set hive.fetch.task.conversion=more;  

select * from emp;  # 不跑MapReduce 特殊场景会?
```

- 本地模式

```shell
# 大多数情况下查询都会触发一个MapReduce任务(job)。Hive中对于某些查询可以不必使用MapReduce，也就是所谓的本地模式
set hive.exec.mode.local.auto=true;

# 当一个job满足如下条件才能真正使用本地模式：
## 1.job的输入数据大小必须小于参数：hive.exec.mode.local.auto.inputbytes.max(默认128MB)
## 2.job的map数必须小于参数：hive.exec.mode.local.auto.tasks.max(默认4)
## 3.job的reduce数必须为0或者1
set hive.exec.mode.local.auto.inputbytes.max=50000000;
set hive.exec.mode.local.auto.tasks.max=10;
```

##### join

- 默认join是inner join
- left join / right join / full join

<img src="pictures/join.png" style="zoom:150%;" />

- 笛卡尔积

```sql
select empno, dname from emp,dept;
select empno, dname from emp CROSS JOIN dept;
```



#### [执行计划](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Explain)

```sql
# 用于调优
explain
select e.empno from emp e;
```



### 4.2.3 内部表 外部表

- 内部表
  - 类型 MANAGED_TABLE
  - 删除表之后：HDFS和MySQL的数据都被删除了（生命周期在Hive内部）

```sql
create table xxx;
```

- 外部表
  - 类型 EXTERNAL_TABLE
  - 删除表之后：MySQL的数据都被删除了，但是HDFS的数据是还存在的（如果数据误删 hdfs上还在）

```sql
create EXTERNAL table xxx;
```

- 相互转换

```sql
-- 改成外部表
ALTER TABLE emp SET TBLPROPERTIES ('EXTERNAL' = 'true');  -- EXTERNAL 必须大写
```

- 应用场景
  - 外部表：比如某个公司的原始日志数据存放在一个目录中，多个部门对这些原始数据进行分析，那么创建外部表是明智选择，这样原始数据不会被删除；
  - 内部表：对原始数据或比较重要的中间数据进行建表存储；
  - 分区表：将每个小时或每天的日志文件进行分区存储，可以针对某个特定时间段做业务分析，而不必分析扫描所有数据；



### 4.2.4 分区表

- 分区表
  - 操作日志表：记录日志、查询操作
  - who when what

- 全表扫描  vs  分区扫描

#### 建立分区表

##### 单分区

```sql
create table dept_partition(
deptno int,
dname string,
loc string
)
PARTITIONED BY (day string)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t';
```

##### 多级分区

```sql
create table dept_partition_d_h(
deptno int,
dname string,
loc string
)
PARTITIONED BY (day string, hour string)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t';
```

#### 导入数据

```sql
-- 单个分区字段
LOAD DATA LOCAL INPATH '/home/hadoop/data/hive/partition/dept_20210201.txt' OVERWRITE INTO TABLE dept_partition PARTITION (day=20210201);

-- 多个分区字段
LOAD DATA LOCAL INPATH '/home/hadoop/data/hive/partition/dept_20210202.txt' OVERWRITE INTO TABLE dept_partition_d_h PARTITION (day='20210202', hour='21');
```

##### insert 导入分区数据

```sql
insert into table dept_partition partition(day='20210203')
select * from dept;
```

#### 修复分区数据

情况：手工创建分区文件夹并用hdfs命令导入数据，无法从hive命令中查到该分区数据？

- 元数据未记录，如果使用load命令会刷新元数据，可以使用msck修复元数据

- msck（ <font color=red>会刷整个分区数据  慎用！！！</font> ）

```sql
-- msck
msck repair table dept_partition;

-- 安全方案
-- step 1 mkdir -p 文件夹
hadoop fs -mkdir -p /user/hive/warehouse/dept_partition/day=20210204
-- step 2 put 上传文件到hdfs
hadoop fs -put partition/dept_20210204.txt /user/hive/warehouse/dept_partition/day=20210204/
-- step 3 分区
ALTER TABLE dept_partition ADD PARTITION(day='20210204');
-- check
show PARTITIONS dept_partition;
```

#### 删除分区

```sql
-- 多个
ALTER TABLE dept_partition DROP PARTITION(day='20210203'),PARTITION(day='20210204');
```

#### 动态分区（上面都是静态分区）

- 建表

```sql
create table emp_dynamic_partition(
  `empno` int, 
  `ename` string, 
  `job` string, 
  `mgr` int, 
  `hiredate` string, 
  `sal` double, 
  `comm` double)
partitioned by(deptno int)  
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t';
```

- 导数
  - 根据deptno 写在select最后一个字段
  - 设置非严格模式nostrict

```sql
-- 将emp表中的数据按照deptno将数据插入到emp对应分区表中
set hive.exec.dynamic.partition.mode=nonstrict;

insert into table emp_dynamic_partition partition(deptno)
select empno,ename,job,mgr,hiredate,sal,comm, deptno from emp;
```

## 4.3 [Function](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+UDF)

### 查看函数

```sql
-- 查看
show functions;

-- details
desc function xxx;
desc function extended xxx;  # example
```

### 时间函数

- unix_timestamp

```sql
-- 时间戳
select unix_timestamp("2008-08-28 13:14:15");

-- 非标准 加yyyyMMdd HHmmss
select unix_timestamp("20080828 131415", "yyyyMMdd HHmmss");
```

- from_unixtime

```sql
select from_unixtime(1607779592);
```

- current_date

```sql
-- 2020-12-12
select current_date;
```

- current_timestamp

```sql
-- 2020-12-12 21:22:46.416
select current_timestamp;
```

- to_date

```sql
select to_date('2009-07-30 04:00:01');
```

- year / month / day / hour / minute / second

```sql
select year('2021-09-01 02:01:03');
select day('2021-09-01 02:01:03');
```

- weekofyear / dayofmonth

```sql
select weekofyear('2021-09-01');
select dayofmonth('2021-09-01');
```

- months_between

```sql
select months_between('2021-03-01', '2021-01-01'); 
```

- add_months
  datediff
  date_add
  date_sub
  last_day  本月最后一天

```sql
select add_months('20210-01-02', 1);
select datediff('2021-04-01', '2021-01-01');
select date_add('2021-01-01', 2);
select last_day('2021-01-01');
```



### [by函数(4个)](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+SortBy)

#### order by

- In the strict mode (i.e., [hive.mapred.mode](https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties#ConfigurationProperties-hive.mapred.mode)=strict), the order by clause has to be followed by a "limit" clause. 
- The limit clause is not necessary if you set <font color=blue>hive.mapred.mode</font> to nonstrict
- 全局排序(默认asc 升序)
- 并且不管来多少数据，都只启动一个reducer来处理。
- 适合数据量小。

#### sort by

-  sort the rows before feeding the rows to a reducer（分区内排序）
-  局部排序
-  sort by会根据数据量的大小启动一到多个reducer来干活
-  并且，它会在进入reduce之前为每个reducer都产生一个排序文件。提高了全局排序的效率

```sql
set mapred.reduce.tasks=3;
-- 查看目录看结果
insert overwrite local directory '/home/hadoop/data/hive/test01' row format delimited fields terminated by '\t' select * from et sort by age;
```

#### distribute by

- 这个不是排序
- 控制map结果的分发，它会将具有相同字段的map（<font color=red>hashCode对reduce个数取模</font>）输出分发到一个reduce节点上做处理
- 某种情况下，我们需要控制某个特定行到某个reducer中，这种操作一般是为后续可能发生的聚集操作做准备

#### cluster by

- 具有distribute by的功能，还会对该字段进行排序
-  如果sort by和 distribute by中所用的列相同，可以改写为cluster by以便同时指向两者所用的列(<font color=blue>分区后再排序</font>)



### 计算型函数

- round
  ceil 向上取整
  floor 向下取整

```sql
select round(2.345, 2);
```

### 字符串

- upper/lower
  length
  trim
  lpad/rpad 指定长度，向左补齐/向右补齐
  regexp_replace
  substr
  concat / concat_ws
  
  split

```sql
select lpad('flink', 10, '-');
# -----flink
# 可以在月份数前补0（<10）

select regexp_replace('2021/01/01', '/', '-'); -- 2021-01-01
select concat_ws('.', '192', '21', '1', '1');  -- 192.21.1.1
select split('192.21.1.1', '\\.');  -- .特殊字符 需要转义
```

- nvl

```sql
nvl(e.deptno,d.deptno)  -- 空替换
```

- collect_set（去重） vs collect_list

### json

- 数据

```json
{"movie":"111", "rate": "3", "time": "424234234", "userid":"1"}
{"movie":"112", "rate": "1", "time": "424234234", "userid":"1"}
```

- json_tuple

```sql
-- create table
create table rating_json(json string);

-- load data
load data local inpath '/home/hadoop/data/hive/rating.json' into table rating_json;

-- parse json table
-- create table t_rate as ...  拷贝数据和表结构
create table t_rate as
select 
  movie, rate, time, userid,
  year(from_unixtime(cast(time as bigint))) as year,
  month(from_unixtime(cast(time as bigint))) as month,
  day(from_unixtime(cast(time as bigint))) as day,
  hour(from_unixtime(cast(time as bigint))) as hour,
  minute(from_unixtime(cast(time as bigint))) as minute,
  from_unixtime(cast(time as bigint)) ts
from (
	select json_tuple(json, "movie","rate","time","userid") as(movie, rate, time, userid) from rating_json ) tmp;
```

- parse_url_tuple

```sql
select parse_url_tuple('http://www.baidu.com/test/film?param1=value1&param2=value2', 'HOST', 'PATH', 'QUERY');
```

### 条件函数

#### if

```sql
select if(1>0,1,0);  -- 1
```

#### case when

```sql
select ename, sal,
  case
  when sal > 1 and sal <= 1000 then 'lower'
  when sal > 1000 and sal <= 2000 then 'm'
  when sal > 2000 and sal <= 3000 then 'high'
  else 'xxx'
  end AS level
from emp;
```

#### 案例

- 数据

```txt
1,PK,RD,1
2,RUOZE,RD,1
3,XIAOHONG,RD,2
4,XIAOZHANG,QA,1
5,XIAOLI,QA,2
6,XIAOFANG,QA,2
```

- 过程

```sql
-- 建表
create table emp_info(
  id string,
  name string,
  dept string,
  sex string
) ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

-- load data
load data inpath '/data/hive/emp_info.txt' into table emp_info;

-- 分组语句 1
select 
  dept, 
  sum(case sex when '1' then 1 else 0 end) m_cnt,
  sum(case sex when '2' then 1 else 0 end) f_cnt
  from emp_info
group by dept;

-- 分组语句 2
select 
  dept, 
  sum(if(sex = '1', 1, 0)) m_cnt,
  sum(if(sex = '2', 1, 0)) f_cnt
  from emp_info
group by dept;
```

### [窗口函数](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+WindowingAndAnalytics)

窗口函数  =  窗口  +  函数

- [文章](https://databricks.com/blog/2015/07/15/introducing-window-functions-in-spark-sql.html)
- 数据

```txt
-- window1.txt
ruozedata,2019-04-10,1
ruozedata,2019-04-11,5
ruozedata,2019-04-12,7
ruozedata,2019-04-13,3
ruozedata,2019-04-14,2
ruozedata,2019-04-15,4
ruozedata,2019-04-16,4

-- window2.txt
dept01,ruoze,10000
dept01,jepson,20000
dept01,xingxing,30000
dept02,zhangsan,40000
dept02,lisi,50000

-- window3.txt
cookie1,2015-04-10 10:00:02,url2
cookie1,2015-04-10 10:00:00,url1
cookie1,2015-04-10 10:03:04,1url3
cookie1,2015-04-10 10:50:05,url6
cookie1,2015-04-10 11:00:00,url7
cookie1,2015-04-10 10:10:00,url4
cookie1,2015-04-10 10:50:01,url5
cookie2,2015-04-10 10:00:02,url22
cookie2,2015-04-10 10:00:00,url11
cookie2,2015-04-10 10:03:04,1url33
cookie2,2015-04-10 10:50:05,url66
cookie2,2015-04-10 11:00:00,url77
cookie2,2015-04-10 10:10:00,url44
cookie2,2015-04-10 10:50:01,url55
```

- 建表

```sql
create table window1(
domain string,
`time` string,
pv int
)ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

create table window2(
dept string,
name string,
salary int
)ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

create table window3(
cookieid string,
time string,
url string
)ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';
```

- load data

```sql
load data local inpath '/home/hadoop/data/hive/window1.txt' into table window1;
load data local inpath '/home/hadoop/data/hive/window2.txt' into table window2;
load data local inpath '/home/hadoop/data/hive/window3.txt' into table window3;
```

#### 应用

- sum/over

```sql
select domain, `time`, pv, 
sum(pv) over(partition by domain order by `time` ) sum1,
sum(pv) over(partition by domain order by `time` ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) sum2,
sum(pv) over(partition by domain order by `time` ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) sum3,
sum(pv) over(partition by domain order by `time` ROWS BETWEEN 3 PRECEDING AND 1 FOLLOWING) sum4,
sum(pv) over(partition by domain order by `time` ROWS BETWEEN 1 PRECEDING AND UNBOUNDED FOLLOWING) sum5
from window1;
```

- NTILE

```sql
-- 切片
-- ntile（2）切2片
select domain, `time`, pv,
  ntile(2) over(partition by domain order by `time`) rn1,
  ntile(3) over(partition by domain order by `time`) rn2
from window1
order by domain, `time`;
```

- row_number / rank / DENSE_RANK

```sql
-- row_number 1 2 3 4 5
-- rank       1 2 3 3 5
-- DENSE_RANK 1 2 3 3 4
select domain, `time`, pv,
  row_number() over(partition by domain order by pv desc) rn1,
  rank() over(partition by domain order by pv desc) rn2,
  DENSE_RANK() over(partition by domain order by pv desc) rn3
from window1;
```

- CUME_DIST / PERCENT_RANK

```sql
-- 小于等于当前值的行数 / 分组内的总行数
select dept, name, salary,
  round(cume_dist() over(order by salary), 2) rn1,
  cume_dist() over(partition by dept order by salary) rn2
from window2;
```

```sql
-- 分组内当前行的rank-1 / 分组内总行数 - 1	
select dept, name, salary,
  round(PERCENT_RANK() over(order by salary), 2) rn1,
  PERCENT_RANK() over(partition by dept order by salary) rn2
from window2;
```

- lead / lag

```sql
-- 从上取
select cookieid, `time`, url,
  lag(`time`, 1, '---') over(partition by cookieid order by `time`) rn1,
  lag(`time`, 2, '---') over(partition by cookieid order by `time`) rn2
from window3;

-- 从下取
select cookieid, `time`, url,
  lead(`time`) over(partition by cookieid order by `time`) rn1,
  lead(`time`, 2, '---') over(partition by cookieid order by `time`) rn2
from window3;
```

- first_value / last_value

```sql
select cookieid, `time`, url,
first_value(url) over(partition by cookieid order by `time`) rn1
from window3;

-- 当前行的最后一行
select cookieid, `time`, url,
last_value(url) over(partition by cookieid order by `time`) rn1
from window3;
```



### UDF

- UDF: 一进一出  upper  lower(ename)
- UDAF: 多进一出   sum  avg
- UDTF：一进多出 

#### UDF 开发

```java
import org.apache.hadoop.hive.ql.exec.Description;
import org.apache.hadoop.hive.ql.exec.UDF;

/**
 * UDF开发步骤：
 * 1) extends UDF
 * 2) 重写evaluate方法
 **/
@Description(name = "say_hello_doc",
        value = "_FUNC_(str) - Returns str with Hello: Ruozedata : str",
        extended = "Example:\n > SELECT _FUNC_('PK') FROM src LIMIT 1;\n 'Hello: Ruozedata-PK'")
public class UDFHello extends UDF {
    public String evaluate(String ename) {
        return  "Hello: Ruozedata-" + ename;
    }

    public String evaluate(String ename, String grade) {
        return  "Hello: Ruozedata-" + grade + " : " + ename;
    }

    public static void main(String[] args) {
        UDFHello udf = new UDFHello();
        System.out.println(udf.evaluate("若泽"));
    }
}
```

- pom.xml

```xml
<dependency>
  <groupId>org.apache.hive</groupId>
  <artifactId>hive-exec</artifactId>
  <version>3.1.2</version>
</dependency>
```

#### 添加UDF方式

##### 临时

```shell
# 方式1
## 只在当前session有效
ADD JAR /home/hadoop/lib/xxxxx-1.0.jar;	
CREATE TEMPORARY FUNCTION say_hello AS "com.hive.UDFHello";

# 方式2 不用每次执行 add jar
mkdir $HIVE_HOME/auxlib  # 存入jar包
CREATE TEMPORARY FUNCTION say_hello AS "com.hive.UDFHello";
```

##### 永久

```sql
# 先通过hdfs上传 再执行
CREATE FUNCTION say_hello AS "com.hive.UDFHello" USING JAR 'hdfs://hadoop000:8020/hive-udf/xx.jar';
```

###### 测试(hive命令行)

```sql
list jars;
select ename, say_hello(ename) from emp;		
```

##### hive启动直接运行（UDF开发2）

- org.apache.hadoop.hive.ql.exec.FunctionRegistry	（hive二次开发）

```java
import org.apache.commons.lang3.StringUtils;
import org.apache.hadoop.hive.ql.exec.UDFArgumentException;
import org.apache.hadoop.hive.ql.exec.UDFArgumentLengthException;
import org.apache.hadoop.hive.ql.metadata.HiveException;
import org.apache.hadoop.hive.ql.udf.generic.GenericUDF;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.primitive.PrimitiveObjectInspectorFactory;

public class UDFLength extends GenericUDF {
    // 初始化
    @Override
    public ObjectInspector initialize(ObjectInspector[] arguments) throws UDFArgumentException {
        if(arguments.length != 1) {
            throw new UDFArgumentLengthException(
                    "UDFLength requires 1 argument, got " + arguments.length);
        }
        return PrimitiveObjectInspectorFactory.javaIntObjectInspector;
    }
    // 处理业务逻辑
    @Override
    public Object evaluate(DeferredObject[] arguments) throws HiveException {
        final String input = arguments[0].get().toString();
        if(StringUtils.isEmpty(input)) {
            return 0;
        }
        return input.trim().length();
    }

    @Override
    public String getDisplayString(String[] children) {
        return null;
    }
}
```



# 5 案例


## 1 行列互转

- 数据

```txt
id,name,dept,sex
1,PK,RD,1
2,RUOZE,RD,1
3,XIAOHONG,RD,2
4,XIAOZHANG,QA,1
5,XIAOLI,QA,2
6,XIAOFANG,QA,2
```

### 行转列

- 数据 

```sql
-- target
相同部门、性别的人合在一起
RD,1 PK|RUOZE
RD,2 XIAOHONG
QA,1 XIAOZHANG
QA,2 XIAOLI|XIAOFANG
```

- 过程

```sql
-- step 1
select 
name, 
concat_ws(',',dept, sex) dept_sex
from 
emp_info;

-- step 2
select
	dept_sex, concat_ws('|', collect_set(t.name))
from
(
  select
  name,
  concat_ws(',',dept, sex) dept_sex
  from
  emp_info) t
group by 
	dept_sex;
```

### 列转行

- explode(col) 
  -  一列中的数据拆分成多行

- 数据

```sql
PK	MapReduce,Hive,Spark,Flink
J	Hadoop,Hbase,Kafka
```

- 过程

```sql
-- create table
create table column2row (
  name string,
  courses string
) row format delimited fields terminated by '\t';

-- load data
load data local inpath '/home/hadoop/data/hive/column2row.txt' into table column2row;

-- 转换
select 
name, course
from
column2row
lateral view explode(split(courses,',')) course_tmp as course;  -- 一进多处
```

## 2 WC统计

- 数据

```txt
hello,hello,hello
world,world
welcome
```

- 过程

```sql
-- table
create table hive_wc(sentence string);

-- load data
hive (default)> load data local inpath '/home/hadoop/data/hive/hive_wc.txt' into table hive_wc;

-- count
select 
word, count(1) cnt
from
(
select
explode(split(sentence,',')) as word 
from
hive_wc
) t
group by word
order by cnt desc;
```

## 3 topN

- 数据

city_info.sql  product_info.sql  user_click.txt

- 建表

```sql
create table user_click(
user_id string,
session_id string,
time string,
city_id int,
product_id int
) partitioned by(day string)
row format delimited fields terminated by ',';	


-- 按照区域求最受欢迎商品的TopN

create table city_info(
city_id int,
city_name string,
area string
)
row format delimited fields terminated by '\t';	


create table product_info(
product_id int,
product_name string,
extend_info string
)
row format delimited fields terminated by '\t';
```

## 4 每个域名截止到每个月的访问数、最大单月访问数、累计到该月总访问数!!!

- 数据

```txt
ruozedata.com	2020-01-02	5
ruozedata.com	2020-01-02	5
ruozedata.com	2020-01-03	15
ruoze.ke.qq.com	2020-01-01	5
ruozedata.com	2020-01-04	8
ruoze.ke.qq.com	2020-01-02	25
ruozedata.com	2020-01-05	5
ruozedata.com	2020-02-01	4
ruozedata.com	2020-02-02	6
ruoze.ke.qq.com	2020-02-01	10
ruoze.ke.qq.com	2020-02-02	5
ruozedata.com	2020-03-01	16
ruozedata.com	2020-03-02	22
ruoze.ke.qq.com	2020-03-01	23
ruoze.ke.qq.com	2020-03-02	10
ruoze.ke.qq.com	2020-03-03	11
```

- 建表

```sql
create table access(
  domain string,
  day string,
  pv int
) row format delimited fields terminated by '\t';
```

```shell
load data local inpath '/home/hadoop/data/hive/access.txt' overwrite into table access;
```

### 1 每个月的访问数

```sql
create table access_tmp1 as
select
domain, date_format(day, 'yyyy-MM') month, sum(pv) pv
from
access
group by domain, date_format(day, 'yyyy-MM');
```

### 2 最大单月访问数

```sql
select 
domain, month, pv,
max(pv) over(partition by domain order by domain,month) max_pv,
sum(pv) over(partition by domain order by domain,month) sum_pv
from 
access_tmp1;
```

### 3 累计到该月总访问数

#### 3.1 自连接

```sql
set hive.exec.mode.local.auto=true;

-- step 1
drop table access_tmp1;
create table access_tmp1 as
select
domain, date_format(day, 'yyyy-MM') month, sum(pv) pv
from
access
group by domain, date_format(day, 'yyyy-MM');

-- step 2 每个域名截止到每个月的访问数、最大单月访问数
select 
domain, month, pv,
max(pv) over(partition by domain order by domain,month) max_pv,
sum(pv) over(partition by domain order by domain,month) sum_pv
from 
access_tmp1;

-- step 3 累计到该月总访问数
-- 从头到当前月
1： 1
2： 1 2
3： 1 2 3
......

drop table access_tmp2;
create table access_tmp2 as
select
  a.domain a_domain, a.month a_month, a.pv a_pv,
  b.domain b_domain, b.month b_month, b.pv b_pv
from
	access_tmp1 a join access_tmp1 b 
on a.domain = b.domain;	

-- 结果
select
  t.b_domain, t.b_month, t.b_pv,
  max(t.a_pv) max_pv,
  sum(t.a_pv) sum_pv
from (
	select * from access_tmp2 where a_month <= b_month
) t 
group by t.b_domain, t.b_month, t.b_pv;
```

#### 3.2 窗口函数

```sql
-- 结果
select 
domain, month, pv,
max(pv) over(partition by domain order by domain,month) max_pv,
sum(pv) over(partition by domain order by domain,month) sum_pv
from 
access_tmp1;
```

- 重复对某个表都N次操作   调优
- 先缓存起来，后面用时 从cache中直接拿
- IO：磁盘  网络	
- 最好的大数据分析方式：忽略掉无关的数据
- 列式存储ORC、Parquet的优势

## 5 消费数

数据

```txt
-- window_orders.txt
pk,2021-09-01,500
xingxing,2021-09-02,3500
pk,2021-02-03,46
xingxing,2021-09-04,578
pk,2021-09-05,345
pk,2021-04-06,235
xingxing,2021-09-07,78
pk,2021-09-08,55
zhangsan,2021-04-08,783
zhangsan,2021-04-09,123
lisi,2021-05-10,150
zhangsan,2021-04-11,456
lisi,2021-06-12,234
zhangsan,2021-04-13,99
```

建表

```sql
create table window_orders(
name string,
time string,
cost double
)row format delimited fields terminated by ',';

load data local inpath '/home/hadoop/data/hive/window_orders.txt' into table window_orders; 
```

#### 每个人累计每月消费数

```sql
select
  name, `time`, cost,
  sum(cost) over(partition by name order by `time`)
from window_orders;
```

#### 连续登陆最多天数

```txt
| pk          | 2021-08-01  |
| pk          | 2021-08-02  |
| pk          | 2021-08-03  |
| pk          | 2021-08-05  |
| rz          | 2021-07-30  |
| rz          | 2021-07-31  |
| rz          | 2021-08-02  |
```

表

```sql
create table login(
  name string,
  time string
)row format delimited fields terminated by ',';

load data local inpath '/home/hadoop/data/hive/login.txt' into table login; 
```

结果

```sql
select name, max(times) max_times
from 
(select name, count(1) times
from (
  select name, time, rn,
		date_sub(time, rn) date_diff
	from
	(select name, `time`,
		row_number() over(partition by name order by time) rn
	from login) t1
) t2 group by name, date_diff) t3
group by name;
```





# 问题

如何控制map个数，reduce个数 ？通过参数，文件大小？

https://mubu.com/doc/1QsdDEluPZd#m
