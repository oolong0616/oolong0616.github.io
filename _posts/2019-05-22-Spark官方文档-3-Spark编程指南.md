---
layout:     post  
title:      笔记：Spark 官方文档(2.2.0)   
subtitle:   Spark编程指南  
date:       2018-05-20  
author:     岑晨  
header-img: 
catalog: true  
tags:  
    - spark
    - 笔记
---  



# 概述 

- 每个Spark应用程序都包含一个**驱动程序**，该*程序*运行用户的`main`功能并在集群上执行各种*并行操作*。Spark提供的主要抽象是***弹性分布式数据集***（RDD），它是跨群集节点分区的元素的集合，可以并行操作。RDD是通过从Hadoop文件系统（或任何其他Hadoop支持的文件系统）中的文件或驱动程序中的现有Scala集合开始并对其进行转换而创建的。用户还可以要求Spark 在内存中*保留* RDD，允许它在并行操作中有效地重用。最后，RDD会自动从节点故障中恢复。

  > - 驱动程序：包含main()函数，Shell就是一个驱动程序
  > - RDD持久化：RDD.persist()或RDD.cache()，其流程是：在action中计算得到rdd；然后，将其保存在每个节点的内存中。

- 在 Spark 中的第二个抽象是能够用于并行操作的 **_shared variables_**（共享变量），默认情况下，当 Spark 的一个函数作为一组不同节点上的任务运行时，它将每一个变量的副本应用到每一个任务的函数中去。有时候，一个变量需要在整个任务中，或者在任务和 driver program（驱动程序）之间来共享。Spark 支持两种类型的共享变量：_broadcast variables_（广播变量），它可以用于在所有节点上缓存一个值，和 _accumulators_（累加器），他是一个只能被 “added（增加）” 的变量，例如 counters 和 sums。

  > Spark是多机器集群部署的，分为Driver/Master/Worker，Master负责资源调度，Worker是不同的运算节点，由Master统一调度。而Driver是我们提交Spark程序的节点，并且所有的reduce类型的操作都会汇总到Driver节点进行整合。节点之间会将map/reduce等操作函数传递一个独立副本到每一个节点，这些变量也会复制到每台机器上，而节点之间的运算是相互独立的，变量的更新并不会传递回Driver程序。
  >
  > - 累加器：是一种只能通过关联操作进行“加”操作的变量，因此它能够高效的应用于并行操作中。它们能够用来实现counters和sums，如果我们需要在spark中进行一些全局统计就可以使用它。
  > - 广播变量：允许程序员缓存一个只读的变量在每台机器上面，而不是每个任务保存一份拷贝。例如，利用广播变量，我们能够以一种更有效率的方式将一个大数据量输入集合的副本分配给每个节点。Spark也尝试着利用有效的广播算法去分配广播变量，以减少通信的成本。       



# 初始化Spark 

创建SparkConf及SparkContext，**SparkConf一旦创建，不可更改**：

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



# 应用发布



