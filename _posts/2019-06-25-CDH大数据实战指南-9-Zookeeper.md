---
layout:     post  
title:      9、Zookeeper    
subtitle:   笔记：CDH大数据平台实战指南 
date:       2019-06-25  
author:     岑晨  
header-img: 
catalog: true  
tags:  
    - CDH 


---

# 概览

包含 leader角色节点和follower角色节点

通过在注册临时类型节点选举leader，当leader节点宕机时，Zookeeper自动删除临时节点，其他节点监听到删除事件后重新争抢注册临时节点