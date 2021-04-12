---
title: Windows系统CMD进程管理
tags: [Windows]
author: Yc-Ma
show_author_profile: true
key: 2021-04-09-Windows系统CMD进程管理
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

## CMD命令
### 进程管理
```
Microsoft Windows [版本 10.0.18362.30]
(c) 2019 Microsoft Corporation。保留所有权利。
```
```
C:\Users\mayc01>netstat -ano|findstr "7475"
```
```
C:\Users\mayc01>netstat -ano|findstr "7425"
  TCP    0.0.0.0:7425           0.0.0.0:0              LISTENING       23456
  TCP    [::]:7425              [::]:0                 LISTENING       23456
```
```
C:\Users\mayc01>kill -9 23456
'kill' 不是内部或外部命令，也不是可运行的程序
或批处理文件。
```
```
C:\Users\mayc01>tasklist|findstr 7425
```
```
C:\Users\mayc01>netstat -ano|findstr "7425"
  TCP    0.0.0.0:7425           0.0.0.0:0              LISTENING       23456
  TCP    [::]:7425              [::]:0                 LISTENING       23456
```
```
C:\Users\mayc01>netstat -ano|findstr "23456"
  TCP    0.0.0.0:7424           0.0.0.0:0              LISTENING       23456
  TCP    0.0.0.0:7425           0.0.0.0:0              LISTENING       23456
  TCP    0.0.0.0:49186          0.0.0.0:0              LISTENING       23456
  TCP    127.0.0.1:49213        127.0.0.1:49214        ESTABLISHED     23456
  TCP    127.0.0.1:49214        127.0.0.1:49213        ESTABLISHED     23456
  TCP    127.0.0.1:49220        127.0.0.1:49221        ESTABLISHED     23456
  TCP    127.0.0.1:49221        127.0.0.1:49220        ESTABLISHED     23456
  TCP    127.0.0.1:49222        127.0.0.1:49223        ESTABLISHED     23456
  TCP    127.0.0.1:49223        127.0.0.1:49222        ESTABLISHED     23456
  TCP    127.0.0.1:49224        127.0.0.1:49225        ESTABLISHED     23456
  TCP    127.0.0.1:49225        127.0.0.1:49224        ESTABLISHED     23456
  TCP    [::]:7424              [::]:0                 LISTENING       23456
  TCP    [::]:7425              [::]:0                 LISTENING       23456
  TCP    [::]:49186             [::]:0                 LISTENING       23456
```
```
C:\Users\mayc01>tasklist|findstr "java"
java.exe                     29448 Console                    2      2,108 K
java.exe                      3260 Console                    2      1,648 K
java.exe                     23456 Console                    2    179,556 K
```
```
C:\Users\mayc01>taskkill /PID 29448 -t -f
成功: 已终止 PID 28088 (属于 PID 29448 子进程)的进程。
成功: 已终止 PID 29448 (属于 PID 18160 子进程)的进程。
```
```
C:\Users\mayc01>tasklist|findstr "java"
java.exe                      3260 Console                    2      1,584 K
java.exe                     23456 Console                    2    179,568 K
```
```
C:\Users\mayc01>taskkill /PID 3260 -t -f
成功: 已终止 PID 16296 (属于 PID 3260 子进程)的进程。
成功: 已终止 PID 3260 (属于 PID 18160 子进程)的进程。
```
```
C:\Users\mayc01>tasklist|findstr "java"
java.exe                     23456 Console                    2    179,592 K
```
```
C:\Users\mayc01>taskkill /PID 23456 -t -f
成功: 已终止 PID 4352 (属于 PID 23456 子进程)的进程。
成功: 已终止 PID 23456 (属于 PID 18160 子进程)的进程。
```
```
C:\Users\mayc01>tasklist|findstr "java"
```
```
C:\Users\mayc01>tasklist|findstr "java"
```
```
C:\Users\mayc01>
```

