---
title: HTTP 80 failed respond
tags: [HTTP]
author: Yc-Ma
show_author_profile: true
key: 2020-10-28-HTTP 80 failed respond
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

# HTTP-80 failed respond

## 报错现象
HTTP访问时出现：
```
2020-10-21 02:04:55.875  WARN 6763 --- [nio-7424-exec-3] n.g.e.SimpleDataFetcherExceptionHandler  : Exception while fetching data (/textDeepAnalyzer) : org.apache.http.NoHttpResponseException: vpc-4zarhbj33zcjjkqfo3afso45la.cn-1.es.alibaba.com.cn:80 failed to respond
```

## 查看HTTP访问设置
![](https://raw.githubusercontent.com/crazyyanchao/blog/master/images/2020/10/2020-10-28-HTTP%2080%20failed%20respond/20201021102704595_17137.png)

## 修改HTTP访问设置【弃用】
- A系统设置的接口为短连接
- B系统设置的接口为长连接
- B系统同一个接口调多次A系统的接口的时候，第一次请求完成之后，A系统会断开连接，而B系统不会断开连接，再次访问A系统的时候还是会用上一次请求的端口继续请求A系统，因为A系统已经释放了连接，所以端口会丢失，再试访问会报错：80 failed to respond
- 设置长、短连接的办法
```
短连接：Connection: close
长连接：Connection: Keep-alive
```

## 修改缓冲区大小
https://stackoverflow.com/questions/26111331/org-apache-http-nohttpresponseexception-xx-xx-xx-xx443-failed-to-respond
```
 /**
  * Finally I fix the issue and it is caused by buffer size. By default, buffer size of httpclient is 8k. So I change it to 4k and my code works well.
  * **/
        ConnectionConfig connectionConfig = ConnectionConfig.custom()
                .setBufferSize(4128)
                .build();
        httpClient = HttpClients.custom()
                .setDefaultConnectionConfig(connectionConfig)
                .build();
        HttpResponse httpResponse = httpClient.execute(httpPost);
```

## 备注
- 【负载过高服务端丢弃连接导致80failed】
- 【HTTP传输层TCP连接复用机制导致80响应失败】
- 【HttpClient的keepalive时间大于服务端，导致服务端关闭连接，可能导致服务端关闭连接超时】
- 【除了在客户端设置小于服务端的Keepalive时间，还有一种做法是在出现NoHttpResponseException时进行重试，也是可以的，还可以减少TIME_WAIT数量。】
- 【修改缓冲区大小】


