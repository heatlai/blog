---
layout: post
title:  '[PHP] Mac + PHP xdebug + Docker + PhpStorm 串接設定'
subtitle: 'PHP - Mac + PHP xdebug + Docker + PhpStorm connect settings'
background: '/img/posts/02.jpg'

date: 2018-11-07

tags: [PHP, Mac, Docker, PhpStorm]
keywords: [xdebug]
---
 
## Mac 
- 增加 loopback IP 給 docker container 吃  
```
$ sudo ifconfig lo0 alias 10.254.254.254 255.255.255.0
```
- 如果是18.03之後可以用特殊DNS，不需設定loopback
```
host.docker.internal
```

## PhpStorm
打開 `Preferences | Languages & Frameworks | PHP | Debug`  
把 `pre-configuration` 做完 然後 `restart phpstorm` 

## Docker 
-  import xdebug.ini 到 php container
```
$ vi ./php/conf.d/xdebug.ini
```
```ini
[xdebug]
xdebug.remote_enable=1
xdebug.profiler_enable=1
xdebug.remote_port=9000
xdebug.remote_autostart=1
xdebug.idekey=PHPSTORM
xdebug.remote_host=host.docker.internal
```
`如果用loopback的話 最後一行替換為 xdebug.remote_host=10.254.254.254`

用 docker run 開的話  
```
docker run -v ./php/conf.d/xdebug.ini:/usr/local/etc/php/conf.d/xdebug.ini
```

用 docker-compose 開的話 docker-compose.yml 加上  
```yaml
volumes:
    - ./php/conf.d/xdebug.ini:/usr/local/etc/php/conf.d/xdebug.ini  
```

## 開網頁
listen 成功的話 開網頁會自動跳出 phpstorm debug server設定  
第一次沒設定到可以到  
phpstorm 的 `Preferences | Languages & Frameworks | PHP | Servers`  
找到 local 用的 domain


