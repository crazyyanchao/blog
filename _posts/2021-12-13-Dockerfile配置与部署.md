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

## 一、Dockerfile配置与部署
- 安装docker命令
```
yum install docker
```
- 打包docker镜像
```
sudo docker build -t graphene:v-1.0.0 .
```
- 查看镜像
```
sudo docker images
```
- 运行镜像
```
sudo docker run -p 8080:8080 graphene:v-1.0.0
```
- 查看docker容器中启动的进程
```
sudo docker ps
```
- 对镜像设置TAG
```
sudo docker tag graphene:v-1.0.0 127.0.0.1/dalaxy/graphene:v-1.0.0
```

```
//修改/etc/hosts
127.0.0.1    harbor.rabyte.com
//修改/etc/docker/daemon.json
{
  "insecure-registry": ["127.0.0.1"],
  "registry-mirrors": ["https://g9bsg11o.mirror.aliyuncs.com", "http://127.0.0.1"]
}
```

```
sudo vim /usr/lib/systemd/system/docker.service
--insecure-registry 127.0.0.1
```
```
sudo docker login 127.0.0.1
model
Model@2021
```
- 提交到镜像服务器
```
sudo docker push 127.0.0.1/dalaxy/graphene:v-1.0.0
```

## 二、Dockerfile配置与部署【测试】
- 安装docker命令
```
yum install docker
```
```
systemctl命令是系统服务管理器指令
启动docker：
systemctl start docker
停止docker：
systemctl stop docker
重启docker：
systemctl restart docker
查看docker状态：
systemctl status docker
开机启动：
systemctl enable docker
查看docker概要信息
docker info
查看docker帮助文档
docker ‐‐help
```
```
sudo docker ps -a
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
FROM node:14.15.1-stretch
ENV NODE_ENV development
# Add a work directory
WORKDIR /graphene
# Cache and Install dependencies
COPY . .
RUN chmod +777 ./node_modules/.bin/webpack
RUN chmod +777 ./node_modules/.bin/webpack-dev-server
#COPY package.json .
RUN npm install
RUN npm run-script build
# Copy app files
# Expose port
EXPOSE 8080
# Start the app
CMD [ "npm", "start" ]
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
