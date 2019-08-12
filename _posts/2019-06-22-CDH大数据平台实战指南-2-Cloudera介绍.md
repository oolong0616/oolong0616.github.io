---
layout:     post  
title:      2、Cloudera介绍    
subtitle:   笔记：CDH大数据平台实战指南 
date:       2019-05-11  
author:     岑晨  
header-img: 
catalog: true  
tags:  
    - CDH   
---

# Cloudera 工具

-  CDH：分发的Hadoop软件发型包，包含Impala和CM
- Impala：Mpp SQL引擎，用于交互式查询分析，适合传统BI查询，可用来查询来自各种源的数据。
- Cloudera Search：搜索近实时访问已存储数据，或者摄取数据到Hadoop或者HBase，提供索引、全文索引和下钻及简单全文检索接口。
- Cloudera Manager（CM）：部署、监控及管理
- Cloudera Navigator：数据管理及监管工具

![Aaron Swartz](https://raw.githubusercontent.com/oolong0616/oolong0616.github.io/master/img/CDH_Frame.png)

# CM 

Hadoop 集群的软件分发及管理监控平台，用于部署Hadoop集群，并对节点及服务进行管理和监控。

![Aaron Swartz](https://raw.githubusercontent.com/oolong0616/oolong0616.github.io/master/img/image-20190623105022762.png)

- Server: 托管 Admin Console Web Server 和应用程序逻辑。 它负责安装软件、配直、启动和停止服务以及管理运行服务的群集。

- Agent:安装在每台主机上 。 它负责启动和停止进程，解压缩配置，触发安装和监控主机。默认情况下， Agent 每隔 15 秒向 Cloudera Manager Server 发送一次检测信号 。 但是，为了减少用户延迟，在状态变化时会提高频率 。如果 Agent 停止检测信号，主机将被标记为运行状况不良。

-  Management Service：执行各种监控、报警和报告功能的一组角色的服务。

- Database ： 存储配置和监控信息 。

- Cloudera Repository：可供 Cloudera Manager 分配的软件的存储库（ repo 库）

  > 用来存放软件安装包？用来存放软件源文件？

- Client： 用于与服务器进行文互的接口 。

  - Admin Console：管理员控制台 。

  - API: Cloudera 产品具有开发的特性，所有在 Cloudera Manager 界面上提供的功能，通过 API都可以完成同样的工作 ， 这些API 都是标准REST API. 开发人员使用 API 甚至可以创建自定义的 Cloudera Manager 应用程序．

    > REST API :http://www.ruanyifeng.com/blog/2011/09/restful.html 

# Cloudera Management Service 

可作为一组角色实施各种管理功能 ：

-  Activity Monitor：收集有关服务运行活动的信息 。
- Host Monitor：收集有关主机的运行状况和指标信息。
- Service Monitor：收集有关服务的运行状况和指标信息 。
- Event Se凹er：聚合组件的事件并将其用于警报和搜索 ．
-  Alert Publisher ：为特定类型的事件生成和提供警报。
-  Repo由 Manager：生成图表报告，提供用户、用户组的目录的磁盘使用率、磁盘 IO 等历史视图 。