---
layout: post
title:  'iThome Cloud Edge Summit Taiwan 台灣雲端大會 2020 筆記'
subtitle:  'iThome Cloud Edge Summit Taiwan 2020 note'
background: '/img/posts/05.jpg'
category: Develop
tags: [Conference]
keywords: [iThome Cloud Edge Summit Taiwan 2020]
---

只記錄有參加到的議程，還有很多沒參加到的議程可以到官網去看簡報。  
- [iThome Cloud Edge Summit Taiwan 2020 官網](https://cloudsummit.ithome.com.tw/)

## 11:20 [101C] serverless 真的方便又省錢？
講者：電商工程師 范建銘  
此議程以 AWS serverless 服務堆疊而成。  
主要訴求
1. 快速解決問題
2. 可以不開主機就不開

自己管主機很麻煩的。  
[簡報](https://s.itho.me/cloudsummit/2020/slides/7005.pdf)
### 案例：大量推播
講者實務經驗，每天五百萬推播(一百萬人一天五則)，每月 100 USD。
```
訂閱
API Gateway ⇢ Lambda ⇢ SNS ( 註冊 User 訂閱 Topic )

推播
API Gateway ⇢ Lambda ⇢ SNS ( Topic 推送 )
                 ⇡
               S3 (推播內容)
```
### 案例：競網爬蟲
被擋 IP 不用怕，AWS IP 擋不完。  
講者實務經驗，每天 2 USD
```
API Gateway ⇢ Lambda ⇢ Lambda ( Multi ) ⇢ Target Website
```
### 案例：自動調整圖片大小 CDN
```
API Gateway ⇢ Lambda(跑縮圖程式) ⇢ S3(縮圖) ⇢  CloudFront(CDN)
                 ⇡
                S3 (原圖)
```
相關資源  
- [AWS 教學課程：搭配使用 AWS Lambda 與 Amazon S3](https://docs.aws.amazon.com/zh_tw/lambda/latest/dg/with-s3-example.html)

### 重點筆記
- API Gateway, Lambda, S3, serverless 三本柱
- 架構好不好看帳單就知道  
- 地端搬上雲端，一比一搬上去是不合理的，應該考慮搬上雲端有什麼好處  
- 不需考慮被雲端商綁死，先把你的服務搞定吧  
- serverless 要看實際需求，不要硬做啊！  
- 注意雲服務限制，尤其是「最多同時執行數」，「每秒最大接收 request 數」,  
  「單次最大執行時間」，「SNS 每秒註冊數」等。

#### 適合場景
- 流量小 ( 因為以次計費 )  
- 執行時間短 ( Lambda 最長執行時間只有 15 分鐘 )  
- 週期性執行  
- 功能簡單，用現成服務可以組出來

## 12:00 [102] 淺談 Spring Cloud Application 轉換至 Kubernetes 的心得
講者：奕兆有限公司 系統架構師 黃健旻  
這個慢慢看😁  
[簡報](https://github.com/Jian-Min-Huang/tech-note/wiki/iThome-Cloud-Edge-Summit-Taiwan-2020)  

## 14:20 [102] Infrastructure as code with Terraform 從零開始導入 Terraform
講者：思華科技 DevOps Engineer 張哲嘉  
[Terraform 官方網站](https://www.terraform.io)  
[投影片](https://slides.com/chechiachang/terraform-introduction)  
[講稿](https://chechia.net/post/terraform-infrastructure-as-code-transcript/)  
[程式碼範例](https://github.com/chechiachang/terraform-playground)  
[SOP 範本](https://github.com/chechiachang/terraform-playground/blob/master/SOP.md)
### 重點筆記
- 只要有手動操作的部分，就算照SOP也可能出錯，不要再手動啦！
- 避免環境調整沒人知道
- 再也不害怕去做環境調整

#### Terraform 三步驟
1. write config  
2. planning 計算變更
3. Apply 套用變更

#### 使用注意
- 透過 Terraform 建立的機器才有 state
- 沒有 state 的機器不會被 Terraform 異動
- 千萬不要用 local state，請用有 lock 的外部 state storage

## 15:10 [101B] DevOps 起手式
講者：格上汽車 租賃數據管理中心經理 周金龍  
[簡報](https://s.itho.me/cloudsummit/2020/slides/7016.pdf)  
### 重點筆記
- 系統整合的問題不在 RESTful 等等東西導入  
- 以 OAuth 和 Microservice 做老中青三代系統串接
- Payment 是做 microservice 的首選
- app ⇢ microservice (會員中心, 自動化授信)，簡報內有系統架構

## 16:00 [102] gRPC：更高效的微服務介面
講者：旋轉拍賣 葉秉哲  
[簡報](https://www.slideshare.net/williamyeh/grpc-238408172/williamyeh/grpc-238408172)  
[gRPC(維基百科)](https://zh.wikipedia.org/wiki/GRPC)
### 重點筆記
- gRPC 全名是 google Remote Procedure Calls，沒錯，又是 google 搞出來的
- HTTP/2 + Protocol Buffers = gRPC
- Header壓縮 + 訊息壓縮及檢驗 (傳輸量越小代表著會越省錢)
- 傳輸更快 (microservice 之間傳輸就是要快)
- 支援 server push
- 外部連線從 API Gateway 進來用 http 1.1，因為還不支援  
- 內部 server 之間連線用 gRPC
- 想要全 gRPC 要等到 http/3 成熟