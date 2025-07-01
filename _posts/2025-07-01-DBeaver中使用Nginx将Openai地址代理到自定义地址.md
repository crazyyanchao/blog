---
title: DBeaver中使用Nginx将Openai地址代理到自定义地址
tags: [Nginx,LLM]
author: Yc-Ma
show_author_profile: true
key: 2025-07-01-DBeaver中使用Nginx将Openai地址代理到自定义地址.md
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}


## DBeaver中使用Nginx将Openai地址代理到自定义地址

```bash
# 安装Nginx For Windows
# 修改hosts，访问域名时保证可以走Nginx代理
127.0.0.1 api.openai.com

# 修改Nginx配置，请看Nginx完整配置

# 加载配置访问映射修改
nginx -p e:\software\nginx-1.29.0 -t
# 使用openssl命令生成 api.openai.crt api.openai.csr api.openai.key 文件
# Postman中测试HTTPS需要从setting中关闭SSL验证

# SSL证书添加到DBEAVER中JRE中`E:\software\dbeaver\jre\lib\security`
& "C:\Program Files\Java\jdk1.8.0_251\bin\keytool.exe" -importcert -alias api-openai-local -file "e:\software\nginx-1.29.0\conf\ssl\api.openai.crt" -keystore "E:\software\dbeaver\jre\lib\security\cacerts" -storepass changeit -noprompt

# 验证证书
& "C:\Program Files\Java\jdk1.8.0_251\bin\keytool.exe" -list -keystore "E:\software\dbeaver\jre\lib\security\cacerts" -storepass changeit | findstr /i api-openai-local
& "C:\Program Files\Java\jdk1.8.0_251\bin\keytool.exe" -list -v -alias api-openai-local -keystore "E:\software\dbeaver\jre\lib\security\cacerts" -storepass changeit

# 强制DBeaver使用指定的trust-store
-Djavax.net.ssl.trustStore=E:\software\dbeaver\jre\lib\security\cacerts
-Djavax.net.ssl.trustStorePassword=changeit
-Djavax.net.ssl.trustStoreType=JKS

# 重试即可成功访问
```

```bash
# Nginx完整配置

#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;
    access_log  logs/access.log  combined;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

# ==============================
# 反向代理 api.openai.com -> api.jsfund.cn
# ==============================
server {
    # 监听 443，并启用 SSL
    listen              443 ssl;
    server_name         api.openai.com;

    # 证书（自己生成或申请后放到 conf/ssl/ 里）
        ssl_certificate     ssl/api.openai.crt;
        ssl_certificate_key ssl/api.openai.key;

    # 一些 TLS 优化（保持默认即可）
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    # 只转发 /v1/chat/completions
    location /v1/chat/completions {
        proxy_pass              http://api.local.cn/dataapi/vest/v1/chat/completions;
        proxy_set_header Host   api.jsfund.cn;       # 让后端看到正确的 Host
        proxy_set_header X-Real-IP        $remote_addr;
        proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
    }

    # 其它路径直接 404，避免误访
    location / {
        return 404;
    }
}

}
```


