---
title: Dockerfile配置与部署
tags: [Docker]
author: Yc-Ma
show_author_profile: true
key: 2021-12-13-Dockerfile配置与部署
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

## Dockerfile配置与部署
- 安装docker命令
```
yum install docker
```
- 打包docker镜像
```
sudo docker build -t ongdb-graphene:v-0.0.1 .
```
- ~~启动docker镜像【本地】~~
```
sudo docker run -p 8080:8080 ongdb-graphene:v-0.0.1
```
- 编辑Dockerfile文件
```
vim Dockerfile
```
```
FROM centos
USER root
WORKDIR /app
ADD . /app
RUN yum install -y npm maven
RUN npm install -g cnpm -registry=https://registry.npm.taobao.org
RUN npm install webpack@4.46.0 webpack-cli@3.3.12 webpack-dev-server@3.11.0 -g
RUN cnpm install clean-webpack-plugin@3.0.0 \
                 css-loader@3.6.0 \
                 file-loader@6.0.0 \
                 html-loader@1.1.0 \
                 html-webpack-harddisk-plugin@1.0.2 \
                 html-webpack-plugin@4.5.2 \
                 mini-css-extract-plugin@0.9.0 \
                 node-sass@4.14.1 \
                 optimize-css-assets-webpack-plugin@5.0.3 \
                 sass-loader@9.0.2 \
                 style-loader@1.2.1 \
                 url-loader@4.1.0 \
                 copy-webpack-plugin@6.2.1 \
                 glob@7.1.6 jquery@3.5.1
WORKDIR /app/browser
#CMD ["cnpm","install"]
#ENTRYPOINT ["cnpm", "start"]
```
- 启动docker镜像【本地】
```
sudo docker run -p 8080:8080 ongdb-graphene:v-0.0.1 bash -c "cnpm install && cnpm start"
```
- 查看docker容器中启动的进程
```
sudo docker ps
```
- 访问服务
```
http://ip:8080/app/
```
