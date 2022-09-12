---
layout: post
title:  '[PHP] 檢查多維陣列內數值是否存在'
subtitle: 'PHP - Check Value Exists Deep In Array'
background: '/img/posts/06.jpg'

date: 2018-05-03

tags: [PHP]
keywords: [deep, array, check, value, exists]
---

# 檢查多維陣列內數值是否存在

```php
function deepInArray( $value, $targetArray )
{
    foreach ( $targetArray as $arrayValue )
    {
        if ( ! is_array( $arrayValue ) )
        {
            if ( $value == $arrayValue )
            {
                return true;
            }
        }
        else
        {
            if ( deepInArray( $value, $arrayValue ) )
            {
                return true;
            }
        }
    }

    return false;
}
```