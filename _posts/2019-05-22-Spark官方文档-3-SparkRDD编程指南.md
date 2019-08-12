---
layout:     post  
title:      3、SparkRDD编程指南    
subtitle:   笔记：Spark 官方文档(更新至2.4.3)  
date:       2018-05-22  
author:     岑晨  
header-img: 
catalog: true  
tags:  
    - spark  
---

# 概述 

- 每个Spark应用程序都包含一个驱动程序(Driver)，该程序运行用户的main功能并在集群上执行各种并行操作。Spark提供的主要抽象是弹性分布式数据集（RDD），它是跨群集节点分区的元素的集合，可以并行操作。RDD是通过从Hadoop文件系统（或任何其他Hadoop支持的文件系统）中的文件或驱动程序中的现有Scala集合开始并对其进行转换而创建的。用户还可以要求Spark 在内存中保留RDD，允许它在并行操作中有效地重用。最后，RDD会自动从节点故障中恢复。

  > - 驱动程序：包含main()函数，Shell就是一个驱动程序
  > - RDD持久化：RDD.persist()或RDD.cache()，其流程是：在action中计算得到rdd；然后，将其保存在每个节点的内存中。

- 在 Spark 中的第二个抽象是能够用于并行操作的_shared variables_（共享变量），默认情况下，当 Spark 的一个函数作为一组不同节点上的任务运行时，它将每一个变量的副本应用到每一个任务的函数中去。有时候，一个变量需要在整个任务中，或者在任务和 driver program（驱动程序）之间来共享。Spark 支持两种类型的共享变量：_broadcast variables_（广播变量），它可以用于在所有节点上缓存一个值，和 _accumulators_（累加器），他是一个只能被 “added（增加）” 的变量，例如 counters 和 sums。


# 初始化Spark 

创建SparkConf及SparkContext，SparkConf一旦创建，不可更改：

```python

conf = SparkConf().setAppName(appName).setMaster(master)
# 创建SparkConf对象
#		appName：应用程序在集群UI上显示的名称
#		master：
  #		local	使用一个工作线程在本地运行Spark（即根本没有并行性）。
  #		local[K]	使用K个工作线程在本地运行Spark（理想情况下，将其设置为计算机上的核心数）。
  #		local[K,F]	使用K个工作线程和F个maxFailures最大重试次数，在本地运行Spark
  #		local[*]	使用与计算机上的逻辑核心一样多的工作线程在本地运行Spark。
  #		local[*,F]	使用与计算机和F maxFailures上的逻辑核心一样多的工作线程在本地运行Spark。
  #		spark://HOST:PORT	连接到给定的Spark独立集群主服务器。端口必须是主服务器配置使用的端口，默认为7077。
  #		spark://HOST1:PORT1,HOST2:PORT2	使用Zookeeper的备用主服务器连接到给定的Spark独立群集。该列表必须具有使用Zookeeper设置的高可用性群集中的所有主主机。端口必须是每个主服务器配置使用的默认端口，默认为7077。
  #		mesos://HOST:PORT	连接到给定的Mesos群集。端口必须是您配置使用的端口，默认为5050。或者，对于使用ZooKeeper的Mesos群集，请使用mesos://zk://...。要提交--deploy-mode cluster，应将HOST：PORT配置为连接到MesosClusterDispatcher。
  #		yarn	根据值的值，以开发模式模式（ --deploy-mode）连接到YARN群集（HADOOP_CONF_DIR YARN_CONF_DIR variable） 
sc = SparkContext(conf=conf)
# 创建SparkContext，此对象可以直接通过读取文件或者实例化内容创建RDD
```

Spark2.0之后，SparkSession出现，SparkSession包含了SQLContext和HiveContext的，其内部封装了SparkContext：

```python
se = SparkSession.builder.config(conf = SparkConf()).getOrCreate() 
	#获取sparkSession： 
sc = se.sparkContext
	#获取SparkContext
```

> appName、Master这种变量不应该在程序中写死，而是应该在调用spark_submit时动态写入

# 应用提交

 `pyspark`调用更通用的`spark-submit`完成程序提交，语法与`spark-submit`保持一致：

```bash
./bin/spark-submit \
  --class <main-class> \
# 执行程序的入口点，主函数，例如org.apache.spark.examples.SparkPi
  --master <master-url> \
# 集群主URL，同创建SparkConf的Master
  --deploy-mode <deploy-mode> \
# 包含两种模式：cluster，client(default)
#   cluster:驱动器程序会被传输并被执行于集群的一个工作节点上
#   client:会将驱动器程序运行在spark-submit被调用的这台机器上。
  --conf <key>=<value> \
# 配置Spark相关属性
  ... # other options
  <application-jar> \
# 包含应用程序和所有依赖项的捆绑jar的路径。URL必须在群集内部全局可见，例如，所有节点上都存在的hdfs://路径或file://路径。
  [application-arguments]
# 需要传递给主函数的方法
```

# RDD

RDD 创建支持两种模式：

- 并行化集合创建：调用SparkContext.parallelize方法，显式创建
- 外部数据集创建 ： 通过读入数据源创建

## 并行化集合创建   

```python
data =[1,2,3,4,6]
rddData =sc.parallelize(data)
# 显式创建RDD
rddData.collect()
# [1, 2, 3, 4, 6]， 显示RDD中所有内容，注意RDD存储内容应可在当前Driver服务器内存中容纳
```

> - Spark 为每个分区提供一个独立的程序
> - 可以通过.parallelize(data,int) 设置分区数量

## 外部数据集创建

```python
textFile =sc.textFile("file:///userDir/spark/spark-2.4.1-bin-hadoop2.6/README.md"，"X")
# 第一参数：
#		本地文件：file:///
# 	HDFS文件：hdfs://
# 	Spark配置SPARK_DIST_CLASSPATH后，默认从HDFS中找文件  
# 	textFile 将文件按照行行切分，存入RDD中
# 第二参数：
#		控制文件分区数，默认为默认情况下，Spark为文件的每个块创建一个分区（HDFS中默认为128MB），但您也可以通过传递更大的值来请求更多的分区，但是分区数不能小于HDFS块数。
textFile.first()
# u'# Apache Spark' 显示第一行
```

注意事项：

- 本地文件系统：文件必须可以被所有工作节点访问，需要把文件拷贝到所有Workers(工作节点)或者网络共享文件系统。
- 支持目录、压缩文件及通配符：例如`textFile("/my/directory")`，`textFile("/my/directory/*.txt")`和`textFile("/my/directory/*.gz")`。       

其他数据格式：

- 目录（SparkContext.wholeTextFiles）：读取包含多个小文件的目录，按照(文件名称,内容）模式返回

- pickle Python对象(`RDD.saveAsPickleFile`和`SparkContext.pickleFile`) ，python中几乎所有的数据类型（列表，字典，集合，类等）都可以用pickle来序列化，pickle序列化后的数据，可读性差，人一般无法识别。

  


> 参照文档  
>     ApacheCN：https://github.com/oolong0616/spark-doc-zh.git  
>     Spark官方：[http://spark.apache.org/docs/2.2.0/rdd-programming-guide.html](http://spark.apache.org/docs/2.2.0/rdd-programming-guide.html)

