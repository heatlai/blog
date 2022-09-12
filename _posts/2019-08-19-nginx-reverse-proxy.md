---
layout: post
title:  '[Nginx] 用 AWS Load Balancer 串接 Nginx 反向代理 PHP-FPM'
subtitle: 'Nginx Reverse Proxy To PHP-FPM With AWS Load Balancer'
background: '/img/posts/01.jpg'
date: 2019-08-19

category: Develop
tags: [Nginx, AWS]
keywords: [Load Balancer, Reverse Proxy, PHP-FPM]
---

# 反向代理在做什麼？
分析 Request 然後轉發至後端不同的 server 進行後續處理   
舉例：  

- "example.com/order" -> "order-server:80"  
- "example.com/sign-in" -> "sign-in-server:80"  
- "example.com/shop" -> "shop-server:80"

對外只有一個 domain，但其實每個不同網址的 Request 是不同 server 在處理

## 架構 
User <- `HTTPS` -> AWS Load Balancer <- `HTTP` -> 反向代理 Nginx <- `HTTP` -> App Nginx <- `fastcgi` -> PHP-FPM

## 踩雷
proxy 通了，但是 URL 打的是 https://xxx.com ，PHP 接收到的 request 卻不是 HTTPS，  
原因是過了 AWS Load Balancer 之後就是 HTTP 連線了，所以需要承接 HTTPS 參數往後送。  
下面 config 是同時需要 HTTP 跟 HTTPS 時才這樣寫，
如果對外服務只允許 https 時，可以寫 force redirect HTTP to HTTPS，  
然後 App Nginx config 寫死 `fastcgi_param HTTPS "on"` 這樣就不需要檢查 X-Forwarded-Proto 了

### AWS Load Balancer Force Redirect HTTP to HTTPS (optional)
```
Listener : HTTP  

[Rule]
IF ： Requests otherwise not routed  
Redirect(301) to https://#{host}:443/#{path}?#{query}
```

### AWS Load Balancer
```
Listener : HTTPS  
Target Group : HTTP  

[Rule]
IF : Host header is "example.com" or "www.example.com" 
THEN : Forward to "Nginx-tier-1"
```

### 反向代理(Reverse Proxy) Nginx Config
```nginx

# 抓取 X-Forwarded 參數繼續後送
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
    
    server_name  www.example.com example.com;
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
    root  /var/www/html/myproject/public;
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




