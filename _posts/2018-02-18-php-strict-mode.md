---
layout: post
title:  '[PHP] 嚴格模式(強型別)'
subtitle: 'PHP - Strict Types'
description: '怎麼開啟 PHP strict_types 嚴格模式(強型別)'
background: '/img/posts/01.jpg'

date: 2018-02-18

tags: [PHP]
keywords: [嚴格模式, Strict Types, strict_types, 強型別]
---

# 嚴格模式(強型別)

原本在弱型別時 hint type 用 int  
當 input 為 (string) '123abc' 時自動轉換為 (int) 123  
造成 input 異常但仍舊繼續執行，不會發生 Exception  
嚴格模式將 PHP 設定為 強型別 檢查  

```php
// 當前檔案有效, 設定時只能在 script 開頭 否則會報錯
declare(strict_types=1);

// 此處 hint type int 改為 string 即正常不會報錯
function testStrict(int $a)
{
    var_dump($a);
}

$str = '123abc';
try
{
    testStrict($str);
}
catch( \TypeError $e)
{
    echo $e->getMessage();
}

// result : Argument 1 passed to testStrict() must be of the type integer, string given
```