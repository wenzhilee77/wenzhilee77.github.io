---
layout: post
title:  Java Manipulate Hadoop FileSystem
categories: Java Manipulate Hadoop FileSystem
tags:  Java Manipulate Hadoop FileSystem
author: wenzhilee77
---

# 依赖 jar 包导入

```java
    <dependencies>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-common</artifactId>
            <version>2.7.7</version>
        </dependency>

        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-hdfs</artifactId>
            <version>2.7.7</version>
        </dependency>
    </dependencies>
```

# 依赖配置文件添加

```java
core-site.xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://192.168.56.101:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/var/hadoop/pseude</value>
    </property>
</configuration>
```

```java
hdfs-site.xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>192.168.56.101:50090</value>
    </property>
    <property>
        <name>dfs.http.address</name>
        <value>192.168.56.101:50070</value>
    </property>
</configuration>
```

# 简单示例程序

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FSDataOutputStream;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import java.net.URI;

public class HDFSDemo {

    private Configuration conf;
    private FileSystem fs;

    @Before
    public void init() throws Exception
    {
        conf = new Configuration();

        //1. jar 版本应与 Hadoop 版本一致
        //2. 设置 user 为 root， 或者用 root 账户启动 jar 包
        fs = FileSystem.get(new URI("hdfs://192.168.56.101:9000"), conf, "root");
    }

    @Test
    public void mkdir() throws Exception
    {
        Path path = new Path("/myTemp");

        if(fs.exists(path))
            fs.delete(path, true);
        fs.mkdirs(path);
    }

    @Test
    public void upload() throws Exception
    {
        Path path = new Path("/myTemp/haha.txt");
        FSDataOutputStream outputStream = fs.create(path);
    }

    @After
    public void close() throws Exception
    {
        fs.close();
    }
}
```