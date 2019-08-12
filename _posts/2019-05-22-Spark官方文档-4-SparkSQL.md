---
layout:     post  
title:      4、SparkSQL指南    
subtitle:   笔记：Spark 官方文档(2.2.0)  
date:       2018-05-24  
author:     岑晨  
header-img: 
catalog: true  
tags:  
    - spark   
---

# 概览  

Spark SQL 是 Spark 处理结构化数据的一个模块，提供了查询结构化书和计算结果的接口，可通过SQL或者DataSet\DataFrame API 访问SparkSQL。

## SQL 接口

- 用途：
  - 执行SQL查询
  - 从Hive中读取数据
- 使用方式:
  - JDBC\ODBC
  - Shell
- 返回值：DataSet or DataFrame

## DataSets 

强类型数据集合，Python中无此API。

## DataFrame

由命名列(Row)组成的DataSet，概念等同于数据库中的表，可通过结构化数据文件、Hive中表、外部数据库或者RDD构建。

# 入门  

## SparkSession 

SparkSession是SparkSQL的入口，通过下述命令构建对象：

```python
from pyspark.sql import SparkSession
# 导入SparkSQL包
spark = SparkSession \
    .builder \
    .appName("Python Spark SQL basic example") \
    .config("spark.some.config.option", "some-value") \
    .getOrCreate()
# appName:设置应用名称，该名称显示在Spark Web UI上
# config：设置相关值

```

> 全部示例：$SPARK_HOME/examples/src/main/python/sql/basic.py

> Spark2.o已提供了对Hive的内置支持，不需额外安装Hive     

## DataFrame
### DataFrame 创建

使用SparkSession，可以从RDD、Hive表或者Spark数据源创建DataFrame。

```python
jf = spark.read.json( \
                     "file:///userDir/spark/spark-2.4.1-bin-hadoop2.6/examples/src/main/resources/employees.json")
jf.show()
###########
# +-------+------+
# |   name|salary|
# +-------+------+
# |Michael|  3000|
# |   Andy|  4500|
# | Justin|  3500|
# |  Berta|  4000|
# +-------+------+
# spark.read下包含所有读入文档函数，包括：
# .jdbc：读取JDBC
#      参数说明：数据库连接url，表名，过滤条件表达式列表，带有用户名密码信息的属性对象。读取了数据之后，形成一个(String,String)对象返回。
#  .json：读取Json文件，需要用前缀HDFS://(默认)或者file://进行标示
# .orc:Optimized Record Columnar，hive数据文件
# .parquet：列式存储数据文件
# .textFile：文本文件
```

### DataFrame操作 

```python
jf.printSchema()
# root
# |-- name: string (nullable = true)
# |-- salary: long (nullable = true)
# 打印DataFrame数据结构信息
jf.select("NAME").show()
# 查询列数据，大小写不敏感
jf.select(jf['name'], jf['salary'] + 1).show()
# +-------+------------+
# |   name|(salary + 1)|
# +-------+------------+
# |Michael|        3001|
# |   Andy|        4501|
# | Justin|        3501|
# |  Berta|        4001|
# +-------+------------+
# 返回([string],[int])
jf.select("name","salary").show()
# +-------+------+
# |   name|salary|
# +-------+------+
# |Michael|  3000|
# |   Andy|  4500|
# | Justin|  3500|
# |  Berta|  4000|
#  +-------+------+
# 返回([string],[string])
jf.filter(jf['salary']>3000).show()
# +------+------+
# |  name|salary|
# +------+------+
# |  Andy|  4500|
# |Justin|  3500|
# | Berta|  4000|
# +------+------+
# 数据过滤
jf.groupby(jf['salary']).count().show()
# 数据分组
# +------+-----+                                                                  
# |salary|count|
# +------+-----+
# |  4500|    1|
# |  4000|    1|
# |  3500|    1|
# |  3000|    1|
# +------+-----+
```

> [Spark SQL API文档]([http://spark.apache.org/docs/2.2.0/api/python/pyspark.sql.html#pyspark.sql.DataFrame](http://spark.apache.org/docs/2.2.0/api/python/pyspark.sql.html#pyspark.sql.DataFrame))

###  SQL语句查询

- Session临时变量

  ```python
  jf.createOrReplaceTempView("employees")
  # 创建或替换本地临时视图。此视图的生命周期依赖于SparkSession类，如果想drop此视图可采用dropTempView删除
  # DataFrame 通过Json文件employees.json文件创建。
  sqlDF=spark.sql("select * from employees")
  # 通过SQL语句筛选视图数据，返回DataFrame，类型与jf中各列对象相同。
  sqlDF.show()
  # 返回DataFrame值
  ```

- Application临时变量

  ```python
  jf.createGlobalTempView("employees")
  # 创建Application级临时视图，此视图在应用范围内可被各Session共享，知道Spark应用程序终止
sqlDF =spark.sql("select * from global_temp.employees")
  # Application临时视图与系统保留数据库global_temp绑定，需要用限定名称引用,发挥一个DataFrame
  sqlDF.show()
  spark.newSession().sql("select * from global_temp.employees")
  ```
  
  > 如果出现报错：Caused by: ERROR XSDB6: Another instance of Derby may have already booted the database /home/hduser/metastore_db.
  > 则删除该目录重新启动，解决该问题
  
## RDD 转换

  Spark SQL支持两种不同的方法将现有RDD转换为数据集。

  - 使用反射来推断包含特定类型对象的RDD的Schema。在你的 Spark 应用程序中当你已知 Schema 时这个基于方法的反射可以让你的代码更简洁。

  - 通过编程接口构造一个Schema，然后将其应用于现有RDD。虽然此方法更详细，但它允许您在直到运行时才知道列及其类型去构造数据集。

> DataFrame中的数据结构信息，即为schema

### 利用反射机制

```python
from pyspark.sql import Row
sc = spark.sparkContext
# 创建SparkContext对象
lines =sc.textFile( \
                   "file:///userDir/spark/spark-2.4.1-bin-hadoop2.6/examples/src/main/resources/people.txt")  
# 读取文件，返回RDD
lines.collect()
# [u'Michael, 29', u'Andy, 30', u'Justin, 19']
# RDD 读取文件与DataFrame读取文件的不同：
# RDD读取文件后，存成一个字符串，DataFrame读取文件后，自动按照行存成多个Row
parts = lines.map(lambda l: l.split(","))
# 把字符串切分成多个数组
parts.collect()
# [[u'Michael', u' 29'], [u'Andy', u' 30'], [u'Justin', u' 19']]
people =parts.map(lambda x:Row(name=x[0],age=int(x[1])))
# 把数组拼接成Row对象，此时，RDD内置格式与DataFrame格式保持一致
schemaPeople = spark.createDataFrame(people)
# 此方法为RDD转换为DataFrame的方法，之前都是在做数据准备
schemaPeople.createOrReplaceTempView("people")
# 注册为Session级临时视图
teenagers = spark.sql(\
                      "SELECT name FROM people WHERE age >= 13 AND age <= 19")
teenNames = teenagers.rdd.map(\
                              lambda p: "Name: " + p.name).collect()
# [u'Name: Michael', u'Name: Andy', u'Name: Justin']
# 返回一个数组
for name in teenNames:
    print(name)
# 循环数组，打印信息
```

### 利用编程接口

Row的内容类型不可确定的情况下，采用此方式，具体步骤包括：

- Setp1：从原始的 RDD 创建 RDD 的 Row（行）。
- Setp2：Step 1 被创建后，创建 Schema 表示一个 StructType匹配 RDD 中的 Row（行）的结构。
- Setp3：通过 SparkSession提供的 createDataFrame 方法应用 Schema 到 RDD 的 RowS（行）

```python
from pyspark.sql.types import *
sc = spark.sparkContext
lines = sc.textFile(\
                    "file:///userDir/spark/spark-2.4.1-bin-hadoop2.6/examples/src/main/resources/people.txt")
# 读取文件，创建RDD
lines.collect()
# [u'Michael, 29', u'Andy, 30', u'Justin, 19']
parts = lines.map(lambda l: l.split(","))
# 拆分成数组
# [[u'Michael', u' 29'], [u'Andy', u' 30'], [u'Justin', u' 19']]
people = parts.map(lambda p: (p[0], p[1].strip()))
# strip():Python 函数，移除首位指定的字符串
# 返回键值对
# [(u'Michael', u'29'), (u'Andy', u'30'), (u'Justin', u'19')]
fields = [StructField(field_name, StringType(), True) \
          for field_name in schemaString.split()]
# 从 schemaString.split()中便利出每个对象，构建以StructField为内容的数组
schema = StructType(fields)
# 封装
# StructType(List(StructField(name,StringType,true),StructField(age,StringType,true)))
schemaPeople = spark.createDataFrame(people, schema)
# 将RDD转换为DataFrame 
schemaPeople.createOrReplaceTempView("people")
# 构建Sesson级临时视图
results = spark.sql("SELECT * FROM people")
results.show()
# +-------+---+
# |   name|age|
# +-------+---+
# |Michael| 29|
# |   Andy| 30|
# | Justin| 19|
# +-------+---+

```
# 数据源

## 通用加载\保存

```python
df = spark.read.load("file:///userDir/spark/spark-2.4.1-bin-hadoop2.6/examples/src/main/resources/users.parquet")
# load()：默认读取.parquet文件，如读取其他格式文件，需指定format
df.show()
# +------+--------------+----------------+                                        
# |  name|favorite_color|favorite_numbers|
# +------+--------------+----------------+
# |Alyssa|          null|  [3, 9, 15, 20]|
# |   Ben|           red|              []|
# +------+--------------+----------------+

```

### 手动指定数据源

```python
df =spark.read.load(\
                    "file:///userDir/spark/spark-2.4.1-bin-hadoop2.6/examples/src/main/resources/people.json", format="json")
# 手动指定数据源
df.select("name", "age").write.save("file:////userDir/spark/spark-2.4.1-bin-hadoop2.6/namesAndAges.json", format="json")
# 保存数据  
# part-00000-56bca6af-1f17-4640-8e9d-e6aefab6ab34-c000.json
# .part-00000-56bca6af-1f17-4640-8e9d-e6aefab6ab34-c000.json.crc
#  _SUCCESS
# ._SUCCESS.crc
# .crc：验证文件
```

### 直接在文件中运行SQL

```python
df = spark.sql("SELECT * FROM parquet.`file:///userDir/spark/spark-2.4.1-bin-hadoop2.6/examples/src/main/resources/users.parquet`")
# 直接用SQL读取文件数据
# 只有 parquet支持此方式
```

### 保存到表

- 即时程序重新启动，持久化表依旧存在

- 基于文件的数据源可以使用如下方法自定义表路径，自定义表路径后，删除表的同时不删除自定义路径，数据将被保留，如不使用自定义表路径，删除时同时删除表路径和数据

  ```python
  df.write.option("path", "/some/path").saveAsTable("t")
  ```

- 持久化数据表可将每个分区的元数据保存到Hive metastore中

  - 可以只返回查询必须的分区，不需要读取所有分区

  - Hive DDL操作可用于Datasource API 创建的表

  - 要同步metastore中分区信息，要调用MSCK REPAIR TABLE 方法，重新读取分区信息。

### 分桶、分区及排序

基于文件的数据源可以支持分桶、分区和排序，只有持久化表可以分桶和排序。

> 分区：初始读入数据不存在分区，分区只作用在<key,value>类型数据上，因此，当一个Job包含Shuffle操作类型的算子时，如groupByKey，reduceByKey etc，此时就会使用数据分区方式来对数据进行分区，即确定某一个Key对应的键值对数据分配到哪一个Partition中。在Spark Shuffle阶段中，共分为Shuffle Write阶段和Shuffle Read阶段，其中在Shuffle Write阶段中，Shuffle Map Task对数据进行处理产生中间数据，然后再根据数据分区方式对中间数据进行分区。最终Shffle Read阶段中的Shuffle Read Task会拉取Shuffle Write阶段中产生的并已经分好区的中间数据。
>
> 分桶：

```python
df.write.bucketBy(42, "name").sortBy("age").saveAsTable("people_bucketed")
# 对输出按照“name"字段分桶，排序并持久化
df.write.partitionBy("favorite_color").format("parquet").save("namesPartByColor.parquet")
# 根据“favorite_color“分区，并保存为parquet格式
```

## Parquet Files

parquet 文件：是Hadoop生态系统中任何项目可用的列式存储格式，其特点为：

- 可以跳过不符合条件的数据，只读取需要的数据，降低 IO 数据量
- 压缩编码可以降低磁盘存储空间（由于同一列的数据类型是一样的，可以使用更高效的压缩编码（如 Run Length Encoding t  Delta Encoding）进一步节约存储空间）
- 只读取需要的列，支持向量运算，能够获取更好的扫描性能
- Parquet 格式是 Spark SQL 的默认数据源，可通过 spark.sql.sources.default 配置
- 当读写该文件时，所有列默认转换为可为空

###  编程形式读入数据

```python
df =spark.read.json(\
                    "file:///userDir/spark/spark-2.4.1-bin-hadoop2.6/examples/src/main/resources/people.json")
# 读取数据
df.write.parquet(\
                    "file:///userDir/spark/spark-2.4.1-bin-hadoop2.6/examples/src/main/resources/people.parquet") 
# 写入parquet文件
dp =spark.read.load(\
                    "file:///userDir/spark/spark-2.4.1-bin-hadoop2.6/examples/src/main/resources/people.parquet")
# 读入parquet文件
dp.createOrReplaceTempView("people")  
# 创建Session级临时视图
spark.sql("select * from people ").show()
```

### 分区解析

​	在一个支持分区的表中，数据是保存在不同的目录中的，并且将分区键以编码方式保存在各个分区目录路径中。Parquet数据源现在也支持自动发现和推导分区信息。例如，我们可以把之前用的人口数据存到一个分区表中，其目录结构如下所示，其中有2个额外的字段，gender和country，作为分区键:

```

path
└── to
    └── table
        ├── gender=male
        │   ├── ...
        │   │
        │   ├── country=US
        │   │   └── data.parquet
        │   ├── country=CN
        │   │   └── data.parquet
        │   └── ...
        └── gender=female
            ├── ...
            │
            ├── country=US
            │   └── data.parquet
            ├── country=CN
            │   └── data.parquet
            └── ...
```

如果需要读取Parquet文件数据，我们只需要把 path/to/table 作为参数传给 SQLContext.read.parquet 或者 SQLContext.read.load。Spark SQL能够自动的从路径中提取出分区信息,只支持数值类型和字符串类型数据作为分区键，随后返回的DataFrame的schema如下：

```
root
	|-- name: string (nullable = true)
	|-- age: long (nullable = true)
	|-- gender: string (nullable = true)
	|-- country: string (nullable = true)
```

从Spark-1.6.0开始，分区发现默认只在指定目录的子目录中进行。以上面的例子来说，如果用户把 path/to/table/gender=male 作为参数传给 SQLContext.read.parquet 或者 SQLContext.read.load，那么gender就不会被作为分区键。如果用户想要指定分区发现的基础目录，可以通过basePath选项指定。例如，如果把 path/to/table/gender=male作为数据目录，并且将basePath设为 path/to/table，那么gender仍然会最为分区键。

### 模型合并

Spark1.5以后禁用

### Hive metastore与Parquet table转换

Spark SQL采用内部Parquet库读取Hive metastore表

#### HIve metastore Parquet 与Parquet table结构（Schema）转换

Hive和Parquet在表结构处理上主要有2个不同点：

- Hive大小写敏感，而Parquet不是
- Hive所有字段都是nullable的，而Parquet需要显示设置

由于以上原因，我们必须在Hive metastore Parquet table转Spark SQL Parquet table的时候，对Hive metastore schema做调整，调整规则如下：

- 两种schema中字段名和字段类型必须一致。调和后的字段类型必须在Parquet格式中有相对应的数据类型，所以nullable是也是需要考虑的。

- 调和后Spark SQL Parquet table schema将包含以下字段：
  - 只出现在Parquet schema中的字段将被丢弃

  - 只出现在Hive metastore schema中的字段将被添加进来，并显式地设为nullable。

#### 元数据刷新

Spark SQL会缓存Parquet元数据以提高性能。如果Hive metastore Parquet table转换被启用的话，那么转换过来的schema也会被缓存。这时候，如果这些表由Hive或其他外部工具更新了，你必须手动刷新元数据。

```python
spark.catalog.refreshTable("my_table")
```

###  配置

Parquet配置可以通过 SQLContext.setConf 或者 SQL语句中 SET key=value来指定

| 属性名                                     | 默认值                                             | 含义                                                         |
| ------------------------------------------ | -------------------------------------------------- | ------------------------------------------------------------ |
| `spark.sql.parquet.binaryAsString`         | false                                              | 有些老系统，如：特定版本的Impala，Hive，或者老版本的Spark SQL，不区分二进制数据和字符串类型数据。这个标志的意思是，让Spark SQL把二进制数据当字符串处理，以兼容老系统。 |
| `spark.sql.parquet.int96AsTimestamp`       | true                                               | 有些老系统，如：特定版本的Impala，Hive，把时间戳存成INT96。这个配置的作用是，让Spark SQL把这些INT96解释为timestamp，以兼容老系统。 |
| `spark.sql.parquet.cacheMetadata`          | true                                               | 缓存Parquet schema元数据。可以提升查询静态数据的速度。       |
| `spark.sql.parquet.compression.codec`      | gzip                                               | 设置Parquet文件的压缩编码格式。可接受的值有：uncompressed, snappy, gzip（默认）, lzo |
| `spark.sql.parquet.filterPushdown`         | true                                               | 启用过滤器下推优化，可以讲过滤条件尽量推导最下层，已取得性能提升 |
| `spark.sql.hive.convertMetastoreParquet`   | true                                               | 如果禁用，Spark SQL将使用Hive SerDe，而不是内建的对Parquet tables的支持 |
| `spark.sql.parquet.output.committer.class` | `org.apache.parquet.hadoop.ParquetOutputCommitter` | Parquet使用的数据输出类。这个类必须是 org.apache.hadoop.mapreduce.OutputCommitter的子类。一般来说，它也应该是 org.apache.parquet.hadoop.ParquetOutputCommitter的子类。注意：1. 如果启用spark.speculation, 这个选项将被自动忽略2. 这个选项必须用hadoop configuration设置，而不是Spark SQLConf3. 这个选项会覆盖 spark.sql.sources.outputCommitterClassSpark SQL有一个内建的org.apache.spark.sql.parquet.DirectParquetOutputCommitter, 这个类的在输出到S3的时候比默认的ParquetOutputCommitter类效率高。 |
| `spark.sql.parquet.mergeSchema`            | `false`                                            | 如果设为true，那么Parquet数据源将会merge 所有数据文件的schema，否则，schema是从summary file获取的（如果summary file没有设置，则随机选一个） |

## Json数据集

> 通常所说的json文件只是包含一些json数据的文件，而不是我们所需要的JSON格式文件。JSON格式文件必须每一行是一个独立、完整的的JSON对象。因此，一个常规的多行json文件经常会加载失败

Spark SQL 可以通过SparkSession.read.load方法自动读取Json数据中结构(Schema)，并保存为DataFrame。

```python
# spark is from the previous example.
sc = spark.sparkContext

# A JSON dataset is pointed to by path.
# The path can be either a single text file or a directory storing text files
path = "file:///examples/src/main/resources/people.json"
peopleDF = spark.read.json(path)

# The inferred schema can be visualized using the printSchema() method
peopleDF.printSchema()
# root
#  |-- age: long (nullable = true)
#  |-- name: string (nullable = true)

# Creates a temporary view using the DataFrame
peopleDF.createOrReplaceTempView("people")

# SQL statements can be run by using the sql methods provided by spark
teenagerNamesDF = spark.sql("SELECT name FROM people WHERE age BETWEEN 13 AND 19")
teenagerNamesDF.show()
# +------+
# |  name|
# +------+
# |Justin|
# +------+

# Alternatively, a DataFrame can be created for a JSON dataset represented by
# an RDD[String] storing one JSON object per string
jsonStrings = ['{"name":"Yin","address":{"city":"Columbus","state":"Ohio"}}']
otherPeopleRDD = sc.parallelize(jsonStrings)
otherPeople = spark.read.json(otherPeopleRDD)
otherPeople.show()
# +---------------+----+
# |        address|name|
# +---------------+----+
# |[Columbus,Ohio]| Yin|
# +---------------+----+
```

## 通过JDBC连接其他数据库

> 以mysql为例

- 下载mysql对应版本Jar文件

  ```bash
  mysql -V
  # 查看mysql版本
  lsb_release -a
  # 查看Ubuntu版本
  # 拷贝对应mysql-connector-java文件到目录,直接放到Spark目录中
  mv mysql-connector-java-5.1.47-bin.jar /userDir/spark/spark-2.4.1-bin-hadoop2.6/jars
  ```

- 配置MySQL支持

  - ~~修改Spark-env.sh(好像不管用)~~
  
- mysql安装测试数据库

  - 下载：https://launchpad.net/test-db/+milestone/1.0.6
  - 解压缩
  - 修改employees.sql

  ```bash
  -- set storage_engine = InnoDB;
  set default_storage_engine = InnoDB;
  -- set storage_engine = MyISAM;
  -- set storage_engine = Falcon;
  -- set storage_engine = PBXT;
  -- set storage_engine = Maria;
  
  -- select CONCAT('storage engine: ', @@storage_engine) as INFO;
  select CONCAT('storage engine: ', @@default_storage_engine) as INFO;
  # 避免 ERROR 1193 (HY000) at line 38: Unknown system variable 'storage_engine'错误
  ```

  - 导入

    ```bash
    mysql -u root -p < employees.sql
    ```
    

- 读取、写入数据

  ```python
  jdbcDF = spark.read\
  							.format("jdbc")\
    						.option("url", "jdbc:mysql://localhost:3306/employees")\
      					.option("dbtable", " employees")\
        				.option("driver","com.mysql.jdbc.Driver")\
     					 .option("user", "root")\
      					.option("password", "1qaz2WSX0plm")\
              	.option("partitionColumn", "emp_no")\
                .option("lowerBound", "10001")\
                .option("upperBound", "400000")\
                .option("numPartitions", "10")\
      					.load()
  # url(读\写)：连接地址
  # dbtable(读\写)：可以使用在SQL查询的 FROM 子句中有效的任何内容。 例如，括号中的子查询。
  # driver：JDBC驱动
  # partitionColumn（读）、lowerBound（读）、 upperBound（读）、numPartitions(读写)
  #		以上四个参数必须同时设置，用与描述读过程中多worker并行访问是数据分区方式。
  #		partitionColumn：分区字段，必须是Num类型
  #   lowerBound：分区字段读取的最小值，j不过滤查询
  #   upperBound：分区字段读取的最大值，不过滤查询
  #   numPartitions：分区数量，收到JDBC链接数量影响
  #   分区规则：（upperBound-lowerBound)/numPartitions确定每个分区数据量，多余数据放在最后一个分区
  # fetchSize（读）：每次读取行数
  # batchsize（写）：每次写入行数
  # isolationLevel（写）：当前连接的事务隔离级别，对应于JDBC's Connection object
  #		NONE：
  #   READ_COMMITTED（默认）：允许不可重复读取，但不允许脏读取。读取数据的事务允许其他事务继续访问该行数据，但是未提交的写事务将会禁止其他事务访问该行。
  #		READ_UNCOMMITTED：允许脏读取，但不允许更新丢失。如果一个事务已经开始写数据，则另外一个数据则不允许同时进行写操作，但允许其他事务读此行数据。
  #		REPEATABLE_READ：禁止不可重复读取和脏读取，但是有时可能出现幻影数据。读取数据的事务将会禁止写事务（但允许读事务），写事务则禁止任何其他事务。
  #		SERIALIZABLE：提供严格的事务隔离。它要求事务序列化执行，事务只能一个接着一个地执行，但不能并发执行。保证新插入的数据不会被刚执行查询操作的事务访问到。
  # truncate（写）：spark可删除已存在的表并且重建，默认false
  # createTableOptions（写）：用来设置创建数据库的特殊参数
  # createTableColumnTypes：用来设置创建表时各字段的类型
  jdbcDF.printSchema()
  #root
  #	|-- emp_no: integer (nullable = true)
  # |-- birth_date: date (nullable = true)
  # |-- first_name: string (nullable = true)
  # |-- last_name: string (nullable = true)
  # |-- gender: string (nullable = true)
  # |-- hire_date: date (nullable = true)
  jdbcDF.show()
  #+------+----------+----------+-----------+------+----------+
  #|emp_no|birth_date|first_name|  last_name|gender| hire_date|
  #+------+----------+----------+-----------+------+----------+
  #| 10001|1953-09-02|    Georgi|    Facello|     M|1986-06-26|
  #| 10002|1964-06-02|   Bezalel|     Simmel|     F|1985-11-21|
  #| 10003|1959-12-03|     Parto|    Bamford|     M|1986-08-28|
  #| 10004|1954-05-01| Chirstian|    Koblick|     M|1986-12-01|
  #| 10005|1955-01-21|   Kyoichi|   Maliniak|     M|1989-09-12|
  #| 10006|1953-04-20|    Anneke|    Preusig|     F|1989-06-02|
  #| 10007|1957-05-23|   Tzvetan|  Zielinski|     F|1989-02-10|
  #| 10008|1958-02-19|    Saniya|   Kalloufi|     M|1994-09-15|
  #| 10009|1952-04-19|    Sumant|       Peac|     F|1985-02-18|
  #| 10010|1963-06-01| Duangkaew|   Piveteau|     F|1989-08-24|
  #| 10011|1953-11-07|      Mary|      Sluis|     F|1990-01-22|
  #| 10012|1960-10-04|  Patricio|  Bridgland|     M|1992-12-18|
  #| 10013|1963-06-07| Eberhardt|     Terkki|     M|1985-10-20|
  #| 10014|1956-02-12|     Berni|      Genin|     M|1987-03-11|
  #| 10015|1959-08-19|  Guoxiang|  Nooteboom|     M|1987-07-02|
  #| 10016|1961-05-02|  Kazuhito|Cappelletti|     M|1995-01-27|
  #| 10017|1958-07-06| Cristinel|  Bouloucos|     F|1993-08-03|
  #| 10018|1954-06-19|  Kazuhide|       Peha|     F|1987-04-03|
  #| 10019|1953-01-23|   Lillian|    Haddadi|     M|1999-04-30|
  #| 10020|1952-12-24|    Mayuko|    Warwick|     M|1991-01-26|
  #+------+----------+----------+-----------+------+----------+
  #only showing top 20 rows
  mid = jdbcDF.filter(jdbcDF['emp_no']>40000)
  # 利用DataFrame清洗数据
  mid.write\
        .format("jdbc")\
        .option("url", "jdbc:mysql://localhost:3306/employees")\
        .option("dbtable", " employees1")\
        .option("driver","com.mysql.jdbc.Driver")\
        .option("user", "root")\
        .option("password", "1qaz2WSX0plm")\
        .save()
   # 写入新表
  mid.write\
        .format("jdbc")\
        .option("url", "jdbc:mysql://localhost:3306/employees")\
        .option("dbtable", " employees")\
        .option("driver","com.mysql.jdbc.Driver")\
        .option("user", "root")\
        .option("password", "1qaz2WSX0plm")\
        .mode("append")\
        .save()
  # saveMode：
  #		error：当将一个DataFrame保存到数据源时，如果数据已经存在，则预期将引发异常。
  #		append：当将一个DataFrame保存到数据源时，如果数据/表已经存在，则希望将DataFrame的内容附加到现有数据中。
  #		overwrite：覆盖模式意味着，当将一个DataFrame保存到一个数据源时，如果数据/表已经存在，则预期现有数据将被DataFrame的内容覆盖。
  #		ignore：忽略模式意味着，当将一个DataFrame保存到一个数据源时，如果数据已经存在，那么save操作将不会保存DataFrame的内容，也不会更改现有的数据。这与SQL中不存在的CREATE表类似。
  ```
  
  > - 客户端会话和执行程序中都应可访问JDBC驱动程序
  > - 注意数据库大小写要求

