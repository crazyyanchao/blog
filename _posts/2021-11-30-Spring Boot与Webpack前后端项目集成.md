---
title: Spring Boot与Webpack前后端项目集成
tags: [Spring Boot,Webpack]
author: Yc-Ma
show_author_profile: true
key: 2021-11-30-Spring Boot与Webpack前后端项目集成
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

## 依赖包安装
### 全局安装package.json中的依赖项
- devDependencies
>在项目目录中安装，其中，--save-dev是本地安装
```
npm install webpack@4.46.0
npm install clean-webpack-plugin@3.0.0
npm install css-loader@3.6.0
npm install file-loader@6.0.0
npm install html-loader@1.1.0
npm install html-webpack-harddisk-plugin@1.0.2
npm install html-webpack-plugin@4.5.2
npm install mini-css-extract-plugin@0.9.0
npm install node-sass@4.14.1
npm install optimize-css-assets-webpack-plugin@5.0.3
npm install sass-loader@9.0.2
npm install style-loader@1.2.1
npm install url-loader@4.1.0
npm install webpack@4.46.0
npm install webpack-cli@3.3.12
npm install webpack-dev-server@3.11.0
```
- dependencies
```
npm install copy-webpack-plugin@6.2.1
npm install glob@7.1.6
npm install jquery@3.5.1
```

- url-loader和fil-loader与图片打包有关，进行安装，安装命令如下
```
#npm install url-loader
#npm install file-loader
```
- -g是全局安装
```
#npm install webpack@4.46.0 webpack-cli@3.3.12 -g
```

```
#（1）第一步
#npm cache clean --force
#（2）第二步 删除node_modules文件夹
#linux上：rm -rf node_modules
window上: 直接手动删除
#（3）如果有package-lock.json文件就删除它，没有不用管，直接跳到第（4）步
3linux上：rm -rf package-lock.json
window上: 直接手动删除
# （4）安装模块
#npm install
```

- cnpm命令安装：`-g`全局安装
```
npm install -g cnpm -registry=https://registry.npm.taobao.org
```

- 本地安装【带入生产环境】
```
cnpm install webpack@4.46.0 webpack-cli@3.3.12
```
```
cnpm install webpack@4.46.0 clean-webpack-plugin@3.0.0 css-loader@3.6.0 file-loader@6.0.0 html-loader@1.1.0 html-webpack-harddisk-plugin@1.0.2 html-webpack-plugin@4.5.2 mini-css-extract-plugin@0.9.0 node-sass@4.14.1 optimize-css-assets-webpack-plugin@5.0.3 sass-loader@9.0.2 style-loader@1.2.1 url-loader@4.1.0 webpack@4.46.0 webpack-cli@3.3.12 webpack-dev-server@3.11.0 copy-webpack-plugin@6.2.1 glob@7.1.6 jquery@3.5.1
```


