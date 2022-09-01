---
layout: post
title:  'Laravel Conf 2019 筆記'
subtitle: 'Laravel Conf 2019 Review'
background: '/img/posts/06.jpg'
description: ""
date: 2019-07-13
category: Develop
tags: [PHP, Laravel]
---

# Laravel Conf 2019 筆記

## Auth0
- 外部認證授權


## 內部微服務 Microservice

### 低耦合

### 高內聚

### 可擴展
- 分離系統功能
auto scaling 可以單獨依照負載量擴展  
例如  
訊息系統 三台  
會員系統 一台  
前台網頁 十台  

### 使用 API 溝通
- 跨語言串接

### 優勢
- 快速開發，只需要開發新系統所需功能
- 彈性版本更新，系統已分離，只需要更換API
- 減少重工
- 加入後續產品開發時程

### 缺點
- 維運困難，完整流程是多服務的集合體，完全依賴API Gateway ，API Gateway 掛掉就全垮
- 安全性問題
- 跨領域考量(網域?)
- 複雜度高，不容易理解系統全貌
- 透過大量API溝通，Latency 成本高
- 整合服務工程師需要研讀所有API文件，還必須即時維護API文件

### 需要注意的事
- 開發 private package
- 收斂參數
- 文件簡單化
- 熟悉行為

## Swoole

### 高併發
- 沒有固定定義，例 C10K
- 通常 讀 > 寫

### 讀

### 寫

### Laravel Performance Tuning
#### Route and Config Files Cache
Cache 流程 : artisan config:cache
Serialized -> Unserialized -> Collection
避免 Cache 同時失效
避免 Cache 穿透(找不到資料，所以沒cache)

Laravel Event -> Queue Driver -> Laravel Queue Worker
Laravel Event -> (RabbitMQ/Redis) -> Laravel Queue Worker

每秒 5 request 

server -> cache -> queue -> db