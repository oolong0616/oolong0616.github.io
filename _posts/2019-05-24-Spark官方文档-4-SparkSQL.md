---
layout:     post  
title:      笔记：Spark 官方文档(2.2.0)    
subtitle:   4、SparkSQL编程指南  
date:       2018-05-20  
author:     岑晨  
header-img: 
catalog: true  
tags:  
		- spark   
		- 笔记
---

# 概览  

Spark SQL 是 Spark 处理结构化数据的一个模块，提供了查询结构化书和计算结果的接口，可通过SQL或者DataSet API 访问SparkSQL。   

## SQL   

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







