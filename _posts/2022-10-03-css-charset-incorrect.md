---
layout: post
title:  '解決 CSS Content 亂碼問題'
subtitle:  'CSS Charset Incorrect'
background: '/img/posts/05.jpg'
category: Develop
tags: [CSS, Browser]
keywords: [charset, incorrect, content, garbled]
---

## 問題
如果 CSS 檔案裡含有 ASCII 編碼以外的 Text 時可能變成亂碼。

舉例常用於 `content`
```css
/* 原始文字 */
a::after { content: "折扣碼"; }
/* 變成亂碼 */
a::after { content: "æŠ˜æ‰£ç¢¼"; }
```

## 分析
檔案確實用 `UTF-8` 編碼儲存，這個沒問題，那就是讀取編碼錯了。

## 編碼判定順序
1. HTTP Response Header 的 `Content-Type`
2. CSS at-rule 的 `@charset`
3. 參考 HTML Document 的編碼
4. 無指定編碼時，假設為 UTF-8

經過測試發現，因為不明原因 charset 應該使用 `UTF-8` 卻變成了 `Windows-1252`。  
那我們就要確定讓其中一條判定成立，至於要讓哪條判定成立，就看環境限制了。

## 解決方案
### 1. HTTP Response Header 的 Content-Type
如果可以修改 Web Server Config 的話，建議採用，有最高優先權。

原本
```text
Content-Type: text/css
```
改為
```text
Content-Type: text/css; charset=utf-8
```

### 2. @charset
無法修改 Web Server Config 時，可採用此方法。
也是比較方便 Developer 的方法。

正確寫法
```css
@charset "UTF-8"; /* 一定要寫在檔案的第一行 */
```
錯誤寫法
```css
@charset 'UTF-8'; /* 不可用單引號 */
@charset  "UTF-8"; /* @charset 跟 "UTF-8" 之間多了一個空格 */
 @charset "UTF-8"; /* @charset 前面多了一個空格  */
@charset UTF-8; /* UTF-8 沒有用引號包起來 */
```

### 3. 參考 HTML Document 的編碼
- `<meta charset>` 元素必須 `完整` 的在 Document 的前 `1024` 個 `bytes` 裡

```html
<html>
    <head>
        <!-- 建議 <head> 和 <meta charset> 之間不要放東西 -->
        <meta charset="utf-8" />
    </head>
</html>
```

### 4. 使用 UTF-8 跳脫文字
- 注意：要移除 unicode 的 `u`，例如 `折` 的編碼是 `\u6298` 要寫成 `\6298` 

```css
/* 原始文字 */
a::after { content: "折扣碼"; }
/* 改為 */
a::after { content: "\6298\6263\78bc"; }
```