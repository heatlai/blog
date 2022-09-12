---
layout: post
title:  'AWS Load Balancer 反向代理 Web App (Nginx + PHP-FPM)'
subtitle: 'AWS Load Balancer Reverse Proxy To Web App (Nginx + PHP-FPM)'
background: '/img/posts/01.jpg'
date: 2019-08-19

category: Develop
tags: [Nginx, AWS]
keywords: [Load Balancer, Reverse Proxy, PHP-FPM]
---

可先閱讀： [什麼是反向代理 (Reverse Proxy)](/post/2019/08/19/reverse-proxy/)

## 架構 
![Architecture Diagram](/img/posts/2019-08-19-reverse-proxy/reverse-proxy-example-aws-elb.png)

### AWS Load Balancer
```
Listener : HTTPS  
Target Group : HTTP  

[Rule]
IF : Host header is "example.com" or "www.example.com" 
THEN : Forward to "TargetGroup"
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

## 踩雷
Reverse Proxy 通了，URL 是 https://example.com ，但 PHP 接收到的 request 卻不是 HTTPS，  
原因是過了 AWS Load Balancer 之後就是 HTTP 連線了，所以需要承接 HTTPS 參數往後送。  
前面的 config 是同時需要 HTTP 跟 HTTPS 時的寫法，
如果對外服務只允許 https 時，可以寫 force redirect HTTP to HTTPS，  
然後 App Nginx config 強制 HTTPS，寫死 `fastcgi_param HTTPS "on"` 這樣就不需要檢查 X-Forwarded-Proto 了

### Nginx Force HTTPS To PHP-FPM
```nginx
# from
fastcgi_param HTTPS $detect_https;
# to
fastcgi_param HTTPS "on"; # 對外只有 HTTPS 時，強制 HTTPS
```

### AWS Load Balancer Force Redirect HTTP to HTTPS (optional)
```
Listener : HTTP  

[Rule]
IF ： Requests otherwise not routed  
Redirect(301) to https://#{host}:443/#{path}?#{query}
```

