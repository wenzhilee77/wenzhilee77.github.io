---
layout: post
title:  "DataX 简介"
categories: DataX
tags:  DataX 数据同步  
author: wenzhilee77
---

* content
{:toc}

## DataX3.0概览
DataX是一个异构数据源(数据库/文件系统)离线同步工具，致力于实现包括关系型数据库(MySQL、Oracle等)、HDFS、Hive、ODPS、HBase、FTP等各种异构数据源之间稳定高效的数据同步功能。

## DataX3.0框架设计
![](https://images2018.cnblogs.com/blog/948447/201807/948447-20180702134246476-1346908980.png)
DataX本身作为离线数据同步框架，采用Framework + plugin架构构建。将数据源读取和写入抽象成为Reader/Writer插件，纳入到整个同步框架中。
Reader：Reader为数据采集模块，负责采集数据源的数据，将数据发送给Framework。
Writer： Writer为数据写入模块，负责不断向Framework取数据，并将数据写入到目的端。
Framework：Framework用于连接reader和writer，作为两者的数据传输通道，并处理缓冲，流控，并发，数据转换等核心技术问题。

## 支持的数据源
DataX目前已经有了比较全面的插件体系，主流的RDBMS数据库、NOSQL、大数据计算系统都已经接入，目前支持数据如下图

| 类型 | 数据源 | Reader(读) | Writer(写) |
| ------ | ------ | ----- | ------ |
| RDBMS 关系型数据库 | MySQL | 支持 | 支持 |
|  | Oracle | 支持 | 支持 |
|  | SQLServer | 支持 | 支持 |
|  | PostgreSQL | 支持 | 支持 |
|  | DRDS | 支持 | 支持 |
|  | 通用RDBMS(支持所有关系型数据库) | 支持 | 支持 |
| 阿里云数仓数据存储 | ODPS | 支持 | 支持 |
|  | ADS |  | 支持 |
|  | OSS | 支持 | 支持 |
|  | OCS | 支持 | 支持 |
| NoSQL数据存储 | OTS | 支持 | 支持 |
|  | Hbase0.94 | 支持 | 支持 |
|  | Hbase1.1 | 支持 | 支持 |
|  | MongoDB | 支持 | 支持 |
|  | Hive | 支持 | 支持 |
|  | 无结构化数据存储 TxtFile | 支持 | 支持 |
|  | FTP | 支持 | 支持 |
|  | HDFS | 支持 | 支持 |
|  | Elasticsearch |  | 支持 |

## DataX3.0核心架构
![](https://images2018.cnblogs.com/blog/948447/201807/948447-20180702134349508-602373442.png)
DataX 3.0 开源版本支持单机多线程模式完成同步作业运行，本小节按一个DataX作业生命周期的时序图，从整体架构设计非常简要说明DataX各个模块相互关系。

### 核心模块介绍：
DataX完成单个数据同步的作业，我们称之为Job，DataX接受到一个Job之后，将启动一个进程来完成整个作业同步过程。DataX Job模块是单个作业的中枢管理节点，承担了数据清理、子任务切分(将单一作业计算转化为多个子Task)、TaskGroup管理等功能。

DataXJob启动后，会根据不同的源端切分策略，将Job切分成多个小的Task(子任务)，以便于并发执行。Task便是DataX作业的最小单元，每一个Task都会负责一部分数据的同步工作。

切分多个Task之后，DataX Job会调用Scheduler模块，根据配置的并发数据量，将拆分成的Task重新组合，组装成TaskGroup(任务组)。每一个TaskGroup负责以一定的并发运行完毕分配好的所有Task，默认单个任务组的并发数量为5。

每一个Task都由TaskGroup负责启动，Task启动后，会固定启动Reader—>Channel—>Writer的线程来完成任务

## 已提供的插件
### Reader插件
* hdfsreader : 支持从hdfs文件系统获取数据。  
* mysqlreader: 支持从mysql数据库获取数据。  
* sqlserverreader: 支持从sqlserver数据库获取数据。 
* oraclereader : 支持从oracle数据库获取数据。   
* streamreader: 支持从stream流获取数据（常用于测试）  
* httpreader : 支持从http URL获取数据。

### Writer插件
* hdfswriter ：支持向hdbf写入数据。  
* mysqlwriter ：支持向mysql写入数据。  
* sqlserverwriter：支持向sqlserver写入数据。  
* oraclewriter ：支持向oracle写入数据。  
* streamwriter ：支持向stream流写入数据。（常用于测试）

## 下载地址
https://github.com/alibaba/DataX

## 环境依赖
* JDK(1.6以上，推荐1.6)
* Python(推荐Python2.6.X)一定要为python2，因为后面执行datax.py的时候，里面的python的print会执行不了，导致运行不成功，会提示你print语法要加括号，python2中加不加都行 python3中必须要加，否则报语法错
* Maven

## 未完待续

......





参考
https://www.cnblogs.com/shujuxiong/p/9253455.html
