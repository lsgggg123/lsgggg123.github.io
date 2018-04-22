---
layout: post
title: '使用nginx将web服务迁移到https'
subtitle: 'HTTPS(Secure Hypertext Transfer Protocol)安全超文本传输协议 它是一个安全通信通道，它基于HTTP开发，用于在客户计算机和服务器之间交换信息。它使用安全套接字层(SSL)进行信息交换，简单来说它是HTTP的安全版。'
date: 2016-09-12
categories: 技术
tags: 技术 运维 nignx linux http 
cover: '/assets/img/20180422/569f26cce4b0e85354ba68da.jpg'
---

HTTPS(Secure Hypertext Transfer Protocol)安全超文本传输协议 它是一个安全通信通道，它基于HTTP开发，用于在客户计算机和服务器之间交换信息。它使用安全套接字层(SSL)进行信息交换，简单来说它是HTTP的安全版。

在互联网安全和个人信息受到空前重视的今天，如果你现在还在使用http的web服务，我建议您尽早做工作升级到https。而且https早已经不是什么新鲜的技术了，现在好多网站已经全面禁止了http，所有的web服务都是走https协议，例如国外的谷歌、facebook、twitter、程序员常用的github、苹果的iOS开发更是强制要求你使用https协议等，国内的知乎网等等都已经全面支持https了。经过我的实践，感觉从http升级到https的过程并不是十分麻烦，所以把这个过程写下来分享给大家。

### 升级思路
第一步要有证书，这一步是最麻烦的。有了证书之后，第二步我们考虑使用nginx的ssl模块来提供https支持，即nginx监听443端口（https协议端口），把请求转发到后端服务器。第三步我们使用nginx监听80端口，把所有请求80的http服务转发到443端口去。最后一步，我们把后端的服务器端口地址（例如tomcat的8080端口）通过防火墙来关掉，只允许nginx转发访问，这样下来，nginx充当了网关的角色，所有的请求都必须经过nginx来处理，而nginx的http也被转到了https上面，这样就实现了整站升级到https，并且省略了后端服务器对https的支持。

![image](/assets/img/20180422/8526b5f6474b4390a7a32aee160c0296.png)

这种方式实现起来较为简单，要求请求从nginx到后端服务器的过程（图中灰色部分）是安全的，绝大部分nginx和后端都是在同一个内网里面的，中间的传输过程可以信任。

### 准备证书

ssl证书的种类有很多，扩展验证型（EV）SSL证书、组织验证型（OV）SSL证书、域名验证型（DV）SSL证书，你甚至还可以自己给自己提供证书，自己颁发的证书不能通过浏览器的认证，所以自己玩玩就好，这里就不介绍了。如果是电商或者金融类的网站，还是建议花钱申请EV证书，当然价格也是不菲的，如果是个人网站，那么DV的证书就足够了。阿里云和腾讯云都提供了一年免费的DV证书申请，非常良心，地址分别是：阿里云和腾讯云，简单的操作之后就可以下载下来申请到的证书，一个crt文件，一个key文件。

### 配置nginx
首先得确定ningx支持ssl模块。可以使用命令nginx -V查看输出的信息有无--with-http_ssl_module的字样，如果没有的话需要重新编译安装一下nginx，加入ssl模块即可。具体可以看这篇文章：[http://my.oschina.net/wojibuzhu/blog/113771](http://my.oschina.net/wojibuzhu/blog/113771)

修改nginx的配置文件，加入下面的配置：

```XML
server {
    listen       443 ssl;
    server_name  www.lsgggg123.com;

    ssl                  on;
    ssl_certificate      /etc/ssl/certificate.crt;
    ssl_certificate_key  /etc/ssl/private.key;

    ssl_session_timeout  5m;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host:443;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        client_max_body_size 10m;
    }
}
```

这样使用https访问就可以反向代理到后端服务器了。

现在http和https是同时支持的，我们接下来要把原来nginx监听的80端口也转到443上面去。配置如下：

```XML
server {
    listen       80;
    server_name  www.lsgggg123.com;
    return       301 https://$server_name$request_uri;
}
```

这样所有的http请求都会返回一个301，客户端会自动再次请求一次https的请求。

### 屏蔽后端服务器的端口

这个比较简单了。使用iptables就可以。

```XML
iptables -I INPUT -p TCP --dport 8080 -j DROP
iptables -I INPUT -s 127.0.0.1 -p TCP --dport 8080 -j ACCEPT
```

最后附一个完整的nginx.conf好了:

```XML
#user  nobody;
worker_processes  1;

error_log  /usr/local/nginx/logs/error.log;

pid        /var/run/nginx.pid;

events {
    use epoll;
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    server {
        listen       80;
        server_name  www.lsgggg123.com;
        return       301 https://$server_name$request_uri;
    }

    server {
        listen       443 ssl;
        server_name  www.lsgggg123.com;

        ssl                  on;
        ssl_certificate      /etc/ssl/cert.crt;
        ssl_certificate_key  /etc/ssl/key.key;
        ssl_session_timeout  60m;

        location ~*\.(js|css)$ {
           root /opt/jars/;
           expires 1d;
        }

        location / {
            proxy_pass http://127.0.0.1:8080;
            proxy_set_header Host $host:443;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            client_max_body_size 10m;
        }
    }
}
```