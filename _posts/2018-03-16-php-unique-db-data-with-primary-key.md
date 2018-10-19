---
layout: post
title:  '[PHP] merge 多個 DB 查詢結果後用 Primary Key 去重複'
subtitle: 'PHP - Unique DB Data with PK'
background: '/img/posts/04.jpg'

date: 2018-03-16

tags: [PHP]
---

有時會用到不同 condition 去查同一個 table 的情況  
merge 多個 result 後又只想留下不重複的 data  

類似下面的樣子
```
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
```
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

以前的菜逼八方法  
foreach 重塞新的array check id existed
```
$uniqueData = array();

foreach ( $dbData as $row )
{
    if( ! isset( $uniqueData[ $row['id'] ] ) )
    {
        $uniqueData[ $row['id'] ] = $row;
    }
}

# uniqueData Gotcha !!

```

解法
```
function uniqueData( array $dbData, string $uniqueKey = 'id' ) : array
{
    return array_values(array_column($dbData, null, $uniqueKey));
}
```