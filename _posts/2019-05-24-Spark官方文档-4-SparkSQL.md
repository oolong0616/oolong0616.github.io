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
    - 笔记  

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



