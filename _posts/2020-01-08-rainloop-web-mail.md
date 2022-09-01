---
layout: post
title:  '[CentOS] Rainloop Mail Web UI 安裝'
subtitle:  'Install Rainloop Mail Web UI on CentOS'
background: '/img/posts/04.jpg'
category: Develop
tags: [CentOS, Mail]
keywords: [Rainloop, Install]
---

# Rainloop

Mail Web UI。  
需要 IMAP & SMTP。  
要開啟「連絡人清單」功能的話還需要 DB。

## 用途
1. 收發信不用安裝App，到處可用。  
2. 可多人使用同一帳號，即時同步信件狀態。  
    例如：support@example.com，給客服人員一起用。
3. 帳號內附掛其他帳號。  
    例如：john@example.com 登入後，可以切換到 support@example.com。

## 安裝
```sh
wget https://www.rainloop.net/repository/webmail/rainloop-community-latest.zip
mkdir /var/www/rainloop
unzip rainloop-community-latest.zip -d /var/www/rainloop

cd /var/www/rainloop
find . -type d -exec chmod 755 {} \;
find . -type f -exec chmod 644 {} \;

# nginx 可能要換成 www-data or apache
chown -R nginx:nginx .
```

## Nginx Config
TLS證書申請參考
[Mail Server設定筆記]({% post_url 2019-11-18-gcp-centos-postfix-dovecot-virtual-domain-tls %}#tls-證書申請)
```nginx
# php-fpm
upstream php-fpm {
    server unix:/run/php-fpm/www.sock;
}

# webmail
server {
    listen       80;
    server_name  webmail.example.com;
    root         /var/www/rainloop;
    location / {
        return 301 https://$server_name$request_uri;
    }
}
server {
    listen       443 ssl;
    server_name  webmail.example.com;
    root         /var/www/rainloop;
    index        index.php;
    charset      utf-8;

    ssl_certificate      /etc/ssl/certs/webmail.cer;
    ssl_certificate_key  /etc/ssl/certs/webmail.key;
    ssl_dhparam          /etc/ssl/certs/dh2048.pem;

    access_log  /var/log/nginx/webmail.access.log main;
    error_log  /var/log/nginx/webmail.error.log;

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    location ~ /\.ht {
        deny all;
    }

    location ^~ /data {
        deny all;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(.*)$;
        fastcgi_keep_conn on;
        include fastcgi_params;
        fastcgi_pass php-fpm;
        fastcgi_index index.php;
        fastcgi_hide_header X-Powered-By;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO       $fastcgi_path_info;
    }
}
```

## 管理者後台
- url : `https://webmail.example.com/?admin`
- username : `admin`
- password : `12345`

1. 語言設定
- `使用者預設語言`「General」->「Interface」->「Language」
- `管理員後台語言`「General」->「Interface」->「Language (admin)」
2. Domain 設定
- 「Login」 -> 「Default Domain」
- 「Domain」 -> 「+ Add Domain」
![Add Domain](/img/posts/2020-01-08-rainloop-web-mail/add_domain.png)
3. 修改管理員帳號密碼
- 「Security」 -> 「Admin Panel Access Credentials」
4. 「連絡人清單」功能
- 「Contacts」 -> 「Enable contacts」
- 「Contacts」 -> 「Storage (PDO)」  
    -- DB 建立一個 rainloop 的 database 就好，table 會自動建立
![Enable contacts](/img/posts/2020-01-08-rainloop-web-mail/enable_contacts.png)