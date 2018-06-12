---
layout: post
title:  '取得陣列內重複的值'
subtitle: 'PHP - Duplicates In Array'
background: '/img/posts/04.jpg'

date: 2018-05-04

categories: development
category: Programing
tags: [PHP]
---

# 取得 array 內重複的 value

```
function duplicatesInArray( array $haystack, bool $strict = false ) : array
{
    return array_unique( array_diff_assoc( $haystack, array_unique( $haystack, $strict ) ), $strict );
}
```