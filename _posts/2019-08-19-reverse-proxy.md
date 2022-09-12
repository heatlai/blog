---
layout: post
title:  '什麼是反向代理 (Reverse Proxy)'
subtitle: 'Reverse Proxy'
background: '/img/posts/01.jpg'
date: 2019-08-19

category: Develop
tags: [Network]
keywords: [Reverse Proxy, Web, Routing, Load Balance]
---

## 用途
1. 路由(Routing)：對外只有一個 Domain，但其實每個不同 URL 的 Request 是不同 Server 在處理
2. 負載平衡(Load Balance)：為後端多個相同的 App Server 進行 Request 分配

## 路由架構
分析 Request URL 然後轉發至後端不同的 Server 進行後續處理

- `https://example.com` -> `web-server:80`  
- `https://example.com/blog` -> `blog-server:80`  
- `https://example.com/admin` -> `admin-server:80`

![Architecture Diagram](/img/posts/2019-08-19-reverse-proxy/reverse-proxy-example-routing.png)

參考： [Nginx 反向代理](/post/2019/08/19/nginx-reverse-proxy/)

## 負載平衡架構
為後端多個相同的 App Server 進行 Request 分配，最常見的分配方式是 Round Robin (循環制)

- `https://example.com` -> [`app-1:80`, `app-2:80`]

![Architecture Diagram](/img/posts/2019-08-19-reverse-proxy/reverse-proxy-example-aws-elb.png)

參考： [AWS Load Balancer 反向代理 Web App (Nginx + PHP-FPM)](/post/2019/08/19/aws-elb-reverse-proxy-to-nginx-php-fpm/)



