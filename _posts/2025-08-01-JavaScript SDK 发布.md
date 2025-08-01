---
title: JavaScript SDK 发布
tags: [JavaScript,NPM]
author: Yc-Ma
show_author_profile: true
key: 2025-08-01-JavaScript SDK 发布
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

## 私有仓库发布
### 配置Home-User目录下的.npmrc文件一

```shell
registry=http://10.0.0.01:8081/repository/npm-internal/
//10.0.27.61:8081/repository/npm-internal/:username=username
//10.0.27.61:8081/repository/npm-internal/:_password=ODhwbmF5THVzd05HQlNmTkp6aDc=  # 原始密码转为Base64
//10.0.27.61:8081/repository/npm-internal/:email=user@email.com
cache=D:\software\nodejs\node_cache
prefix=D:\software\nodejs\node_global
strict-ssl=false
```

### 配置Home-User目录下的.npmrc文件二

```shell
registry=https://registry.npmjs.org/
always-auth=true
//registry.npmjs.org/:_authToken=<***>
```

## 打包
```
npm install
npm build
```

## 上传
```
npm publish
```

