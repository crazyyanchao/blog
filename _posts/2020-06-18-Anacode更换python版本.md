---
title: Anacode更换python版本
tags: [Tag1,Tag2]
---

Here's the table of contents:
1. TOC
{:toc}

## 使用以下命令创建新环境
```
// 其中 -n 代表 name，env_name 是需要创建的环境名称，list of packages 则是列出在新环境中需要安装的工具包
conda create -n env_name list of packages
```
## 安装python3.6
```
conda create -n py36 python=3.6
```
## 激活这个新配置的环境
```
conda activate py36
```
## 查看python版本
```
python --version
```
## 删除配置的新环境
```
conda env remove -n env_name
```
## 显示所有环境
```
conda env list
```
## 当前环境下的package信息存入名为environment的YAML文件中
```
conda env export > environment.yaml
```
## 配置相应的环境【用对方分享的 YAML 文件来创建一摸一样的运行环境】
```
conda env create -f environment.yaml
```

