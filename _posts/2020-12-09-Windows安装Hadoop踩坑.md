---
title: Windows安装Hadoop踩坑
tags: [Hadoop,Windows]
author: Yc-Ma
show_author_profile: true
key: 2020-12-09-Windows安装Hadoop踩坑
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

### 报错信息
```
E:\workspace\ongdb\halin>hadoop version
系统找不到指定的路径。
Error: JAVA_HOME is incorrectly set.
       Please update E:\software\ongdb-spark\hadoop-2.7.7\conf\hadoop-env.cmd
'-Xmx512m' 不是内部或外部命令，也不是可运行的程序
或批处理文件。
```

### 修改配置
- E:\software\ongdb-spark\hadoop-2.7.7\etc\hadoop
```
set JAVA_HOME=C:\PROGRA~1\Java\jdk1.8.0_251
```
```
【PROGRA~1】表示【Program Files】文件名
```

### 配置成功
```
E:\workspace\ongdb\halin>hadoop version
Hadoop 2.7.7
Subversion Unknown -r c1aad84bd27cd79c3d1a7dd58202a8c3ee1ed3ac
Compiled by stevel on 2018-07-18T22:47Z
Compiled with protoc 2.5.0
From source with checksum 792e15d20b12c74bd6f19a1fb886490
This command was run using /E:/software/ongdb-spark/hadoop-2.7.7/share/hadoop/common/hadoop-common-2.7.7.jar
```

