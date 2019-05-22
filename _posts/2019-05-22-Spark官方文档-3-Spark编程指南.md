---
layout:     post  
title:      笔记：Spark 官方文档(2.2.0)   
subtitle:   Spark编程指南  
date:       2019-05-20  
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

  > - 变量：

