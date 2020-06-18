---
title: python报错unable to import 'smart_open.gcs', disabling that module处理
tags: [Python]
---

Here's the table of contents:
1. TOC
{:toc}

## 按顺序安装模块
安装顺序是依次安装numpy+kml、scipy、gensim，根据自己Python下载的版本进行下载
如果你的库里面有numpy、scipy,请卸载后安装！
[下载地址](https://www.lfd.uci.edu/~gohlke/pythonlibs/)

## 下载的模块示例
``
gensim-3.8.3-cp37-cp37m-win_amd64.whl
numpy-1.16.6+mkl-cp37-cp37m-win_amd64.whl
scipy-1.4.1-cp37-cp37m-win_amd64.whl
smart_open-1.10.0-py3-none-any.whl
``
## 卸载命令
```
python -m pip uninstall smart_open
python -m pip uninstall scipy
python -m pip uninstall gensim
python -m pip uninstall numpy
```
## 安装命令
```
python -m pip install D:\software\python\numpy-1.16.6+mkl-cp37-cp37m-win_amd64.whl
python -m pip install D:\software\python\scipy-1.4.1-cp37-cp37m-win_amd64.whl
python -m pip install D:\software\python\gensim-3.8.3-cp37-cp37m-win_amd64.whl
python -m pip install D:\software\python\smart_open-1.10.0-py3-none-any.whl
```
## 备注
执行'import gensim'如果看到以下报错，则卸载后重新按顺序安装
```
import Error:DDL load failed
```
## Oracle报错处理
### 报错内容1
```
cx_Oracle.DatabaseError: DPI-1047: Cannot locate a 64-bit Oracle Client library: "The specified modu
```
- 安装Oracle
```
pip install cx_Oracle
```
- 下载instant-client将其下所有OCI.dll文件复制到site-packages下面【C:\Users\mayc01\Anaconda3\Lib\site-packages】
[下载instant-client](https://www.oracle.com/database/technologies/instant-client/winx64-64-downloads.html)
- 查看site-packages目录
```
import site; site.getsitepackages()
```

### 报错内容2
```
sqlalchemy.exc.DatabaseError: (cx_Oracle.DatabaseError) ORA-12154: TNS: 无法解析指定的连接标识符
```
- [下载OCI](https://www.oracle.com/database/technologies/instant-client/winx64-64-downloads.html)
- 将OCI放置在【C:\Users\mayc01\Anaconda3\Lib\site-packages】目录下

### 报错内容3
```
sqlalchemy.exc.DatabaseError: (cx_Oracle.DatabaseError) DPI-1072: the Oracle Client library version is unsupported
```
- [下载cx-Oracle 5.3](https://pypi.org/project/cx-Oracle/5.3/#files)
```
cx_Oracle-5.3-11g.win-amd64-py3.6-2
```

