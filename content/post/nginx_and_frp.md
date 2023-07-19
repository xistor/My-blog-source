---
title: "Nginx和Frp共用80/443端口"
date: 2023-07-17T23:12:00+08:00
tags: ["nginx"]
categories: ["network"]
---

一般来说frp在代理http的时候会占用80/443端口，但是这样太浪费了，想要在vps上再跑一个nginx，按照如下配置实测可以实现。

### 网络结构

{{< figure src="/img/nginx-and-frp/network-topology.png"  class="center" title="网络拓扑">}}


网络包括一台有公网ip的vps和一台内网的Linux小主机， 外网共用80/443端口，通过Nginx根据子域名分流。 vps上跑着一个web， 同样的内网小主机上也有几个web服务，内网使用http访问，外网通过frp及https2http插件转成https 穿透到外网访问。  

域名和ssl证书会用到，参考其他资料获取。

### FRP配置

frps 服务端配置

```ini
# [common] is integral section
[common]
# A literal address or host name for IPv6 must be enclosed
# in square brackets, as in "[::1]:80", "[ipv6-host]:http" or "[ipv6-host%zone]:80"
bind_addr = 0.0.0.0
bind_port = 4567
# udp port used for kcp protocol, it can be same with 'bind_port'
# if not set, kcp is disabled in frps
kcp_bind_port = 4567

vhost_http_port = 8088
vhost_https_port = 5566
# console or real logFile path like ./frps.log
log_file = ./log/frp/frps.log
# debug, info, warn, error
log_level = debug
log_max_days = 3
# auth token
privilege_mode = true
token = xxxxxx
# It is convenient to use subdomain configure for http、https type when many people use one frps server together.
subdomain_host = xistor.top
# only allow frpc to bind ports you list, if you set nothing, there won't be any limit
#allow_ports = 1-65535
# pool_count in each proxy will change to max_pool_count if they exceed the maximum value
max_pool_count = 50
# if tcp stream multiplexing is used, default is true
tcp_mux = true
```

`vhost_http_port` 默认是`80` ，改为其他，`vhost_https_port` 默认是`443`改为其他。重启frps， 之后frps会监听这两个端口。


frpc配置

参考之前的[笔记](https://blog.xistor.top/post/home-server-config)


### Nginx 配置


修改 `/etc/nginx/conf.d/default.conf` , 新建一个server监听8443端口，这就是vps上自己的https web服务的端口。

```
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /var/www/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /var/www/html;
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


server {
    listen 8443 ssl;
    server_name  localhost;

    ssl_certificate          /root/certs/xistor.top.cer;
    ssl_certificate_key      /root/certs/xistor.top.key;

    ssl_session_timeout  5m;

    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers   on;

    location / {
        root   /var/www/html;
        index  index.html index.htm;
    }
}

```
修改`/etc/nginx/nginx.conf`, 在底部添加如下配置，根据子域名转发到对应端口，这里以nextcloud所在的子域名next为例。其他的就默认转到vps的web服务上。


```
...
stream {
    map $ssl_preread_server_name $name {
        next.xistor.top frp; #映射你frp用到的域名到合适的后端名frp
        default web; #未匹配到任何域名时遵从此服务
    }
    upstream frp {
        server 127.0.0.1:5566; #这里设置frp监听的用于映射https的端口为上游
    }
    upstream web {
        server 127.0.0.1:8443; #改成前面的8443
    }
    server {
        listen 443 reuseport; #nginx监听的对外的443端口
        proxy_pass $name; #反代
        ssl_preread on; #预读sni主机名
    }
}

```


现在应该就可以通过`https://next.xistor.top`访问内网的nextcloud了，并且通过 `https://xistor.top` 也可以访问vps上的web服务。  

可以再加一个配置，强制使用https访问nextcloud。新建一个`/etc/nginx/conf.d/frp.conf`

```
server {
    listen 80;
    server_name next.xistor.top;
    rewrite ^(.*)$  https://$host$1 permanent;
    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

### Nextcloud 配置

经过层层代理转发后，在nextcloud 登录时， 会出现跳转到`127.0.0.1`的现象，需要添加新的信任代理到`config.php`中

```php

'trusted_proxies'   => ['127.0.0.1'],
'overwritehost'     => 'next.xistor.top',

```





### 参考：
https://shiping.date/frp_https_site.html  
https://cengelsen.no/en/blogg/nextcloud-instruks  
https://help.nextcloud.com/t/solved-nextcloud-15-redirect-to-local-ip/45352/3