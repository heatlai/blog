---
layout: post
title:  '[PHP] merge 多個 DB 查詢結果後用 Primary Key 去重複'
subtitle: 'PHP - Unique DB Results with Primary Key'
background: '/img/posts/03.jpg'

date: 2018-03-16

tags: [PHP]
keywords: [unique, array, db results, primary key, merge]
---

有時會用到不同 condition 去查同一個 table 的情況  
merge 多個 results 後又不想要重複的 record  

類似下面的樣子
```php
$dbData = [
    [
        'id' => 1,
        'name' => 'Heat',
    ],
    [
        'id' => 2,
        'name' => 'John',
    ],
    [
        'id' => 1,
        'name' => 'Heat',
    ],
    [
        'id' => 2,
        'name' => 'John',
    ]
];

```

想去重複變成
```php
$dbData = [
    [
        'id' => 1,
        'name' => 'Heat',
    ],
    [
        'id' => 2,
        'name' => 'John',
    ]
];
```

以前的菜鳥方法  
foreach check id 不在新 array 裡後 set value
```php
$uniqueData = array();

foreach ( $dbData as $row )
{
    if( ! isset( $uniqueData[ $row['id'] ] ) )
    {
        $uniqueData[ $row['id'] ] = $row;
    }
}

$uniqueData = array_values($uniqueData);

# uniqueData Gotcha !!

```

解法
```php
function uniqueData( array $dbData, string $uniqueKey = 'id' ) : array
{
    return array_values(array_column($dbData, null, $uniqueKey));
}
```