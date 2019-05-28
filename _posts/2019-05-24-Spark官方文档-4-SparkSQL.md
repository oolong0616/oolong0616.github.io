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
jf=spark.read.json("file:///userDir/spark/spark-2.4.1-bin-hadoop2.6/examples/src/main/resources/employees.json")
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

###  SQL查询

