---
layout: post
title:  '[PHP] SnakeCase CamelCase 轉換'
subtitle: 'PHP - SnakeCase CamelCase Transform'
background: '/img/posts/05.jpg'

date: 2018-05-02

tags: [PHP]
keywords: [SnakeCase, CamelCase, Transform]
---

# SnakeCase CamelCase 互轉

```php
function snakeCaseToCamelCase( string $string, bool $capitalizeFirstCharacter = false ) : string
{

    $str = str_replace('_', '', ucwords($string, '_'));

    if ( ! $capitalizeFirstCharacter )
    {
        $str = lcfirst($str);
    }

    return $str;
}

function camelCaseToSnakeCase( string $string ) : string
{
    return strtolower(preg_replace(['/([a-z\d])([A-Z])/', '/([^_])([A-Z][a-z])/'], '$1_$2', $string));
}
```