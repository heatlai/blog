---
layout: post
title:  '[Nginx] 反向代理 + 負載平衡'
subtitle: 'Nginx Reverse Proxy + Load Balancer'
background: '/img/posts/01.jpg'
date: 2019-08-19

category: Develop
tags: [Nginx]
keywords: [Load Balancer, Reverse Proxy, PHP-FPM]
---

可先閱讀： [什麼是反向代理 (Reverse Proxy)](/post/2019/08/19/reverse-proxy/)

## 架構
![Architecture Diagram](/img/posts/2019-08-19-nginx-reverse-proxy/reverse-proxy-example-load-balancer.png)

### 反向代理(Reverse Proxy) Nginx Config
```nginx
# 抓取 X-Forwarded 參數繼續往後送 (如果有)
map $http_x_forwarded_proto $x_proto {
    default $http_x_forwarded_proto;
    "" "https";
}
map $http_x_forwarded_port $x_port {
    default $http_x_forwarded_port;
    "" "443";
}

# 後端 app server 群組
upstream app_group 
{
    server app-1:80;
    server app-2:80;
}

server {
    listen       80;
    listen       [::]:80;
    
    server_name  example.com www.example.com;
    root         /usr/share/nginx/html;

    location / {
        proxy_pass http://app_group;
        proxy_redirect default;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        
        # 轉送 load balancer 資訊到後端
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-Proto $x_proto;
        proxy_set_header X-Forwarded-Port $x_port;
    }

    error_page 404 /404.html;
    location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
    }
}
```

### App(PHP-FPM) Nginx Config
```nginx

# 抓取 X-Forwarded-Proto 參數 檢查是否為 https
map $http_x_forwarded_proto $detect_https {
    default "";
    https "on";
}

server {
    listen       80;
    listen       [::]:80;

    server_name  example.com www.example.com;
    root  /app/public;
    index index.php index.html index.htm;
    charset utf-8;
    server_tokens off;

    fastcgi_hide_header X-Powered-By;

    error_page 404 /index.php;
    
    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass php-fpm:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        
        # 傳送 https 參數給 PHP
        fastcgi_param HTTPS $detect_https;
    }
}
``` 




