---
title: Python SDK 发布
tags: [Python,PyPI]
author: Yc-Ma
show_author_profile: true
key: 2024-08-05-Python SDK 发布
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

## 私有仓库发布
### 配置Home-User目录下的.pypirc文件

```shell
[distutils]
index-servers =
    nexus

[nexus]
repository = http://localhost:8080/repository/py_release/
username = username
password = password
```

### 打包
```python
del /f /q dist\*.*
# 打包前确认已经配置了setup.py文件
python setup.py sdist
# 或使用uv
uv build
```

### 上传
```
pip install twine
twine upload -r nexus dist\*
```

## 开源仓库发布

### 确认已经配置setup.py文件
```
# 与本地发布类似
```

### 在pypi挂网注册账号
```
https://pypi.org/
```

### 生成 API token
```
https://pypi.org/manage/account/
```

### 在github仓库设置token
```
# 新增Repository secrets
https://github.com/crazyyanchao/llmcompiler/settings/secrets/actions
Repository secrets
```

### 配置yml文件
```shell
# 在项目目录下生成.yml文件
# 参考如下
# .github/workflows/python-publish.yml
name: Publish Python package to PyPI

on:
  push:
    tags:
      - 'v*.*.*'
    branches:
      - main
  release:
    types: [ published ]
  workflow_dispatch:

jobs:
  publish:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install setuptools wheel twine

      - name: Build package
        run: |
          python setup.py sdist bdist_wheel

      - name: Publish package
        env:
          TWINE_USERNAME: __token__
          TWINE_PASSWORD: ${{ secrets.PYPI_API_TOKEN }}
        run: |
          python -m twine upload dist/*
```

### 参考项目
[LLMCompiler](https://github.com/crazyyanchao/llmcompiler)

