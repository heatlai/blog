---
layout: post
title:  '[CentOS] Fail2ban 防暴力破解'
subtitle: 'Fail2ban - prevent Brute-force attack on CentOS 7'
background: '/img/posts/03.jpg'
category: Develop
tags: [CentOS, Mail, Security]
keywords: [Fail2ban, Prevent, Brute-Force Attack]
---

# Fail2Ban

## 用途
監聽 `Service Log File` 找出想`封鎖`的 `Client`，  
可以用在`防暴力破解密碼`，`掃網址`之類的行為  
例如:
- 2分鐘內，登入失敗100次，就擋掉60分鐘
- 一直寄信給不存在的信箱地址, 像是 vivian@example.com
- 一直被 try 不存在的 uri, 像是 example.com/editWhiteList

## 環境
- CentOS 7

## 安裝
```bash
systemctl enable --now firewalld
yum install -y epel-release
yum install -y fail2ban
```

## 設定
確認設定，不一定需要修改
```bash
vi /etc/fail2ban/fail2ban.conf

# 修改 log path
logtarget = /var/log/fail2ban.log
```
```bash
# jail.conf 裡面的 [DEFAULT] 是 banaction = iptables-multiport
# 在 CentOS 7 要用 firewallcmd-ipset
vi /etc/fail2ban/jail.d/00-firewalld.conf

[DEFAULT]
banaction = firewallcmd-ipset[actiontype=<multiport>]
banaction_allports = firewallcmd-ipset[actiontype=<allports>]
```

### 實例 [ Mail Service 防暴力破解 ]
- postfix-sasl
- dovecot

```bash
vi /etc/fail2ban/jail.local

[DEFAULT]
# 不想擋的IP or CIDR mask
ignoreip = 127.0.0.1 192.168.0.0/16

# 2分鐘內失敗100次就擋掉，m = 分鐘
findtime = 2m
maxretry = 100

# 每次要擋多久 (秒)，m = 分鐘
bantime = 60m

# 要監聽的服務
[postfix-sasl]
enabled = true

[dovecot]
enabled = true
# 因為 dovecot 不會寫 log 到 systemd
# 所以 backend 不能使用預設的 systemd 且要指定 logpath
backend = auto
logpath = /var/log/dovecotlog
```

## 啟動
```bash
systemctl enable --now fail2ban
```

## 設定檔說明
##### 設定檔讀取順序 <small>(`後面`的設定檔會`覆蓋前面`的設定檔)</small>
1. /etc/fail2ban/jail.conf
2. /etc/fail2ban/jail.d/*.conf, alphabetically
3. /etc/fail2ban/jail.local
4. /etc/fail2ban/jail.d/*.local, alphabetically

有關 `postfix-sasl` 的 `DEFAULT` 設定，  
可以在 `/etc/fail2ban/jail.conf` 找到下面設定
```bash
[postfix-sasl]
filter   = postfix[mode=auth]
logpath  = %(postfix_log)s
backend  = %(postfix_backend)s
```
分別是  
- `filter` : 在 log file 裡要尋找的 pattern,  
    設定檔路徑是 `/etc/fail2ban/filter.d`  
    根據上面的設定，設定檔為 `/etc/fail2ban/filter.d/postfix.conf` 並輸入參數 `mode=auth`
- `logpath` : log file 路徑  
- `backend` : 監聽模式，通常設 `auto` 即可
  
那麼 `%(postfix_log)s` 和 `%(postfix_backend)s` 又是什麼鬼呢？  
`%(postfix_log)s` 是命名為 `postfix_log` 的 `變數`  
這些`變數`在哪裡`定義`的呢？
在 `/etc/fail2ban/jail.conf` 裡面會看到
```bash
[INCLUDES]
before = paths-fedora.conf
```
看 `[INCLUDES]` 往上追，在 `/etc/fail2ban/paths-fedora.conf` 裡又看到
```bash
[INCLUDES]
before = paths-common.conf

[DEFAULT]
syslog_mail_warn = /var/log/maillog
postfix_backend = systemd
```
這邊就看到 `%(postfix_backend)s` 的設定值是 `systemd`  
再往上追，在 `/etc/fail2ban/paths-common.conf` 裡又可以看到
```bash
[DEFAULT]
postfix_log = %(syslog_mail_warn)s
```
發現 `postfix_log` 是 `%(syslog_mail_warn)s`  
也就是在 `paths-fedora.conf` 裡定義的 `syslog_mail_warn = /var/log/maillog`

## Fail2ban 是怎麼擋這些 client 的
可以看到前面在 `/etc/fail2ban/jail.d/00-firewalld.conf` 的設定
```bash
[DEFAULT]
banaction = firewallcmd-ipset[actiontype=<multiport>]
banaction_allports = firewallcmd-ipset[actiontype=<allports>]
```
ban 掉 client 要用的是 `firewallcmd-ipset`  
那這個 `action` 在哪裡定義的呢？  
action設定檔路徑 `/etc/fail2ban/action.d/`  
在 `/etc/fail2ban/action.d/firewallcmd-ipset.conf` 裡面
```
actionstart = ipset create <ipmset> hash:ip timeout <bantime><familyopt>
               firewall-cmd --direct --add-rule <family> filter <chain> 0 <actiontype> -m set --match-set <ipmset> src -j <blocktype>

actionban = ipset add <ipmset> <ip> timeout <bantime> -exist
```
`actionstart` 是第一次 action 的動作，  
上面定義要新建一個 `ipset` 並且 `firewalld` 要讀取這個新建的 `ipset` 設定，  
`actionban` 是發現符合ban條件時的動作，  
是增加 `IP` 和 `expire time` 進 `ipset`

## 狀態檢查
### 檢查 ban 狀態
```bash
fail2ban-client status postfix-sasl
# print
Status for the jail: postfix-sasl
|- Filter
|  |- Currently failed:	7
|  |- Total failed:	3645
|  `- Journal matches:	_SYSTEMD_UNIT=postfix.service
`- Actions
   |- Currently banned:	2
   |- Total banned:	655
   `- Banned IP list:	1.1.1.1 2.2.2.2
```

### 手動解除封鎖 IP
```bash
fail2ban-client set postfix-sasl unbanip 1.1.1.1
```

### firewalld 檢查 
###### 在 `Banned IP list` 有東西之後才會有，還沒有人被 `Ban` 之前看不到
```bash
firewall-cmd --direct --get-all-rules

# 會看到已經設定 `--match-set f2b-postfix-sasl` 採取 `REJECT` 行為
ipv4 filter INPUT_direct 0 -p tcp -m multiport --dports smtp,465,submission,imap,imaps,pop3,pop3s -m set --match-set f2b-postfix-sasl src -j REJECT --reject-with icmp-port-unreachable
```
### ipset 檢查
###### 在 `Banned IP list` 有東西之後才會有，還沒有人被 `Ban` 之前看不到
```bash
ipset list

# print
Name: f2b-postfix-sasl
Type: hash:ip
Revision: 4
Header: family inet hashsize 1024 maxelem 65536 timeout 600
Size in memory: 120
References: 1
Number of entries: 0
Members:
```

### 查看 Log
狀態說明：
- Unban：`bantime` 過期所以`解除封鎖`的 IP
- Ban：`開始封鎖`的IP
- Found：`登入失敗`的IP，失敗次數就是用這個算
```bash
cat /var/log/fail2ban.log
# print
2019-12-23 11:29:23,529 fail2ban.actions        [4085]: NOTICE  [postfix-sasl] Unban 123.123.123.123
2019-12-23 11:29:47,357 fail2ban.filter         [4085]: INFO    [postfix-sasl] Found 123.123.123.123 - 2019-12-23 11:29:47
2019-12-23 11:29:55,576 fail2ban.actions        [4085]: NOTICE  [postfix-sasl] Ban 1.1.1.1
```
