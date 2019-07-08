---
layout:     post  
title:      Centos安装Mysql    
subtitle:   笔记
date:       2018-11-19  
author:     岑晨  
header-img: 
catalog: true  
tags:  
    - Mysql     
    - Centos     


---

# Centos7 离线安装Mysql 5.6

- 卸载Mariadb

  ```bash
  rpm -qa | grep mysql #查看MySQL安装情况
  rpm -qa|grep -i mariadb # 查看mariadb安装情况
  rpm -qa|grep mariadb|xargs rpm -e --nodeps #卸载mariadb
  rpm -qa|grep -i mariadb # 确认是否卸载
  
  ```

  > 在新版本的CentOS7中，默认的数据库已更新为了Mariadb，而非 MySQL，所以执行 yum install mysql 命令只是更新Mariadb数据库，并不会安装 MySQL 。

- 下载安装包：https://dev.mysql.com/downloads/mysql/5.6.html#downloads，安装RedHat版本

- 解压

  ```bash
  tar -xvf XXXX.tar
  ```

- 安装

  - 依赖安装

    ```bash
    yum install perl
    yum -y install autoconf 
    yum -y install numactl
    yum install  libaio-devel.x86_64
    yum install nc 
    nc -l -p 3306
    ```

  - 安装

    ```bash
    rpm -ivh MySQL-client-5.6.44-1.el7.x86_64.rpm --force --nodeps # 安装客户端
    rpm -ivh MySQL-devel-5.6.44-1.el7.x86_64.rpm --force --nodeps # 安装开发包
    rpm -ivh MySQL-server-5.6.44-1.el7.x86_64.rpm --force --nodeps # 安装服务端
    rpm -ivh MySQL-shared-5.6.44-1.el7.x86_64.rpm --force --nodeps 
    ```

  - 启动服务器

    ```bash
    service mysql start
    ```

  - 查看状态

    ```bash
    service mysql status
    ```

  - 修改初始化密码

    - 寻找my.cnf文件

      ```bash
      find / -name my.cnf
      ```

    - 修改my.cnf文件，设置无密码登入

      ```bash
      skip-grant-tables # 加入
      ```

    - 重启Mysql

      ```bash
      systemctl restart mysql 
      ```

    - 无密码登录（如不成功，再重启一下服务）

      ```bash
      mysql -u root -p 
      ```

    - 设置

      ```sql
      use mysql;
      update user set password=password('1qaz2WSX') where user='root' and host='localhost';
      commit;
      flush privileges; 
      ```

    - 删除my.cnf文件修改内容

    - 重新设置密码

      ```sql
      SET PASSWORD = PASSWORD('1qaz2WSX');
      commit;
      flush privileges; 
    ```
  
- 授权其他机器登录
  
  - 配置my.cnf
  
      ```bash
      bind-address = 0.0.0.0
      port=3306
    ```
  
  - 设置数据库
  
      ```sql
      use mysql;
      GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '1qaz2WSX' WITH GRANT OPTION;
      FLUSH  PRIVILEGES;
      update user set host='%' where user='root' and host='localhost';  # 报错不用理会
      FLUSH  PRIVILEGES;
    ```
  
    
  
    