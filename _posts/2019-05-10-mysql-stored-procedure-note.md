---
layout: post
title:  '[MySQL] Stored Procedure 筆記 & 範例'
subtitle: 'MySQL - Stored Procedure Note & Example'
background: '/img/posts/05.jpg'
description: ""
date: 2019-05-10
category: Develop
tags: [MySQL, Database, RDBMS]
keywords: [Stored Procedure]
---

# Stored Procedure
Stored Procedure 是在資料庫中建立的一組工作程序，
個人理解就是把不想在 App Server 上跑的部分，寫成 Function 在 DB 裡跑。
有些需求會反覆用到 DB，例如做統計報表、遞迴查詢，或是不想把邏輯放在App，例如用戶登入，
都可以寫一個 Stored Procedure 一次完成要執行的工作。

## 範例：用戶登入
### 用OUT拿回傳值
`需要一次 exec， 一次 query 拿回傳值`

- MySQL Stored Procedure

```sql
DROP PROCEDURE IF EXISTS `user_login`;
DELIMITER //
CREATE PROCEDURE `user_login`(IN in_username varchar(20),IN in_password varchar(20),OUT out_user_id INT)
main:BEGIN
    SELECT max(`user_id`) INTO out_user_id
    FROM `users` 
    WHERE `username` = in_username and `password` = in_password;
END
//
DELIMITER ;
```

- PHP 取結果

```php
<?php

// 用戶登入的帳密
$username = 'userA';
$password = 'userA_password';

// DB 連線
$pdo = new PDO("mysql:host=127.0.0.1;dbname=testing", 'root', '@@mysql');

// 執行 sql
$q = $pdo->exec("CALL user_login('$username','$password',@user_id)");
$res = $pdo->query('SELECT @user_id AS user_id')->fetchAll(PDO::FETCH_ASSOC);
print_r($res);

?>
```

### 直接回傳值
`一次 query 就拿回傳值，這個比較方便`

- MySQL Stored Procedure

```sql
DROP PROCEDURE IF EXISTS `user_login`;
DELIMITER //
CREATE PROCEDURE `user_login`(IN in_username varchar(20),IN in_password varchar(20))
main:BEGIN
    SELECT max(`user_id`) AS user_id
    FROM `users` 
    WHERE `username` = in_username and `password` = in_password;
END
//
DELIMITER ;
```

- PHP 取結果

```php
<?php

// 用戶登入的帳密
$username = 'userA';
$password = 'userA_password';

// DB 連線
$pdo = new PDO("mysql:host=127.0.0.1;dbname=testing", 'root', '@@mysql');

// 執行 sql
$res = $pdo->query("CALL user_login('$username','$password')")->fetchAll(PDO::FETCH_ASSOC);
print_r($res);

?>
```

## 範例：統計報表
`計算結果後寫入另一張表`

- MySQL Stored Procedure

```sql
DROP PROCEDURE IF EXISTS `count_city_shops`;
DELIMITER //
CREATE PROCEDURE `count_city_shops`()
main:BEGIN
    DECLARE p_total_city INT;
    DECLARE p_city_id INT;
    DECLARE p_shop_count INT;
    DECLARE p_today CHAR(10);
    
    DECLARE cursorCityShops CURSOR FOR
           SELECT c.city_id,
                  sum(CASE WHEN s.shop_id IS NOT NULL THEN 1 ELSE 0 END) AS shop_count
           FROM citys c
           LEFT JOIN shops s ON c.city_id = s.city_id
           GROUP BY c.city_id;
       
    SELECT COUNT(*) INTO p_total_city FROM citys;
    SELECT DATE_FORMAT(NOW(),'%Y-%m-%d') INTO p_today;
   
    SET @cityIndex = 0;
    OPEN cursorCityShops;
    WHILE p_total_city > @cityIndex DO
        FETCH cursorCityShops INTO p_city_id,p_shop_count;
        
        INSERT INTO count_shop_table(city_id, shop_count, created_date)
            VALUES (p_city_id, p_shop_count, p_today);
        
        SET @cityIndex = @cityIndex + 1;
    END WHILE;
    CLOSE cursorCityShops;
END
//
DELIMITER ;
```

- PHP 執行

```php
<?php

// DB 連線
$pdo = new PDO("mysql:host=127.0.0.1;dbname=testing", 'root', '@@mysql');

// 執行 sql
$res = $pdo->exec("CALL count_city_shops()");
print_r($res);

?>
```

## 範例：查詢父層級(遞迴查詢)
- 測試資料
```sql
DROP TABLE IF EXISTS `categories`;

CREATE TABLE `categories` (
  `id` int(11) NOT NULL,
  `parent_id` int(11) DEFAULT NULL,
  `name` varchar(45) DEFAULT NULL,
  `order` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

INSERT INTO `categories` VALUES (0,NULL,'Root',NULL),(1,0,'Food',NULL),(2,1,'Fast food',NULL),(3,2,'American',NULL),(4,2,'Japan',NULL),(5,3,'McDonald',NULL),(6,4,'Ichran',NULL);
```
- MySQL Stored Procedure
```sql
DROP PROCEDURE IF EXISTS `getParent`;
DELIMITER //
CREATE PROCEDURE `getParent`(
  IN `typeId` int,
  OUT `result` varchar(255)
)
BEGIN
  DECLARE `parentTypeId` int;
  DECLARE `currentTypeName` varchar(255);
  SELECT `parent_id`, `name` INTO `parentTypeId`, `currentTypeName`
  FROM categories
  WHERE `id` = `typeId`;

  IF `parentTypeId` IS NOT NULL THEN
    CALL `getParent`(`parentTypeId`, `result`);
    SET `result` := CONCAT( IFNULL(`result`, ''), ' > ', `currentTypeName`);
  ELSE
    SET `result` := CONCAT( IFNULL(`result`, ''), `currentTypeName`);

  END IF;

END//

DELIMITER ;

call getParent( 5, @result);

select @result;
```
