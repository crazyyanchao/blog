---
title: CentOS系统安装Python3环境
tags: [CentOS,Cypher,Python,Pandas]
author: Yc-Ma
show_author_profile: true
key: 2021-11-01-CentOS系统安装Python3环境
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

### 安装Python3
```
yum install python36
```
### 安装pip工具
```
sudo yum install python-pip
sudo pip3 install --upgrade pip
```
### 安装cypher包
```
pip3 install ipython-cypher
pip3 install pandas
```
### 执行Python文件
```
python3 test.py
```
### CentOS后台运行Python
```
nohup python3 -u test.py > test.log 2>&1 &
```


