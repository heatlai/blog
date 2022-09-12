---
layout: post
title:  '[JavaScript] 取得根網域'
subtitle: 'JavaScript - Get Root Domain (Super Domain)'
background: '/img/posts/03.jpg'

date: 2018-06-11

keywords: [js, root domain, super domain, set-cookie]
tags: [JavaScript, Cookie]
---

當有多個子網域 (subdomain) 時方便在 JavaScript 裡 set-cookie  
此解法是抓取目前網址，從 Domain 最後一段開始測試是否 set-cookie 成功，如果成功就是 Root Domain  
舉例在網址 `promo.example.com.tw` 時的測試順序，：
- ".tw" `失敗`
- ".com.tw" `失敗`
- ".example.com.tw" `成功，這個就是 Root Domain`

## Code

```javascript
window.ROOT_DOMAIN = (function() {
    let hostname = document.location.hostname.split('.');

    if( hostname.length === 1)
    {
        return hostname[0];
    }

    let tryCookie = 'try_get_root_domain=cookie';
    
    for (let i = hostname.length - 1; i >= 0; i--)
    {
        let tryingName = hostname.slice(i).join('.');
        document.cookie = tryCookie + ';domain=.' + tryingName + ';';

        if (document.cookie.indexOf(tryCookie) > -1)
        {
            document.cookie = tryCookie.split('=')[0] + '=;domain=.' + tryingName + ';expires=Thu, 01 Jan 1970 00:00:01 GMT;';
            return tryingName;
        }
    }
})();
    
window.ROOT_DOMAIN; // return "example.com.tw" 
```