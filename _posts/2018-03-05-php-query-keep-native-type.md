---
layout: post
title:  '[PHP] SQL Query 保留原始型別'
subtitle: 'PHP - SQL Query Keep Native Types'
description: 'PHP 的 SQL Query result 保留原始型別，讓 int 不會變成 string'
background: '/img/posts/02.jpg'

date: 2018-03-05

tags: [PHP]
keywords: [mysqli, PDO, Keep Native Type, integer return as integer]
---

PDO 跟 mysqli 都預設 fetch 回來的 value 皆為 String  
要拿回 numeric ( int 或 float ) 需要加上 option 才能保持數字型別

### 準備
```php
$host = '127.0.0.1';
$port = 3306;
$dbType = 'mysql';
$dbName = 'testing';
$username = 'root';
$passwd = '@@mysql';
```

### mysqli
```php
$mysqli = new mysqli($host, $username, $passwd, $dbName, $port);
$mysqli->options(MYSQLI_OPT_INT_AND_FLOAT_NATIVE, 1);
```

### PDO
```php
$dsn = "{$dbType}:dbname={$dbName};charset=utf8mb4;host={$host};port={$port}";
$pdo = new PDO($dsn, $username, $passwd);
$pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);  // 必要，預設 : true
$pdo->setAttribute(PDO::ATTR_STRINGIFY_FETCHES, false); // 非必要，預設 : false，但還是都加上吧
```