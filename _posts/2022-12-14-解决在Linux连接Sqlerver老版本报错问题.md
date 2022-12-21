---
title: 解决在Linux连接Sqlerver老版本报错问题
tags: [Linux,SQLServer]
author: Yc-Ma
show_author_profile: true
key: 2022-12-14-解决在Linux连接Sqlerver老版本报错问题
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

## 报错
```
The server selected protocol version TLS10 is not accepted by client preferences [TLS12]
```

## 修复
- 找到Java安装目录
```
which $JAVA_HOME
```
- 修改jre/lib/security目录下的java.security文件
```
/usr/lib/jvm/java-1.8.0/jre/lib/security
```
```
# Example:
#jdk.tls.disabledAlgorithms=MD5, SSLv3, DSA, RSA keySize < 2048
jdk.tls.disabledAlgorithms=SSLv3, RC4, MD5withRSA, DH keySize < 768
#jdk.tls.disabledAlgorithms=SSLv3, TLSv1, TLSv1.1, RC4, DES, MD5withRSA, \
#    DH keySize < 1024, EC keySize < 224, 3DES_EDE_CBC, anon, NULL, \
#    include jdk.disabled.namedCurves
```


