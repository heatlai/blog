---
layout: post
title:  '[CentOS] Mail Server + 虛擬帳號 + Mailgun on GCP 設定筆記'
subtitle: 'CentOS 7 + Postfix + Dovecot + Virtual User + Mailgun Setup on GCP'
background: '/img/posts/02.jpg'
description: "手把手建置一台 CentOS Mail Server on GCP"
date: 2019-11-18
category: Develop
tags: [GCP, CentOS, Mail]
keywords: [Postfix, Dovecot, Mailgun]
---

# Mail Server 設定筆記
- OS : CentOS 7
- SMTP : postfix port 25, 465, 587
- POP3 : dovecot port 995
- IMAP : dovecot port 993
- 虛擬帳號設定 : 虛擬網域 & 虛擬使用者
- GCP 鎖 port 25，只能用第三方寄信服務，我選用 Mailgun
- DKIM 跟 SPF 設定去看 Mailgun 文件，這兩個東西我沒有自己做:D
- 後面有附加一些方便操作的 Shell Script，用的時候要用 root 或 sudo

## DNS 設定
### A record
```text
mail.example.com A xxx.xxx.xxx.xxx
```
### MX record
```text
example.com MX 10 mail.example.com
```

## 台北時區

```bash
timedatectl set-timezone Asia/Taipei
# or
ln -sf /usr/share/zoneinfo/Asia/Taipei /etc/localtime
```

## 關閉 ipv6
`開 ipv6 可能會遇到 DNS反解 的問題`
- localhost

```bash
vi /etc/hosts

# 註解下面 ipv6 這行
#::1         localhost

:wq
```
- 網卡

```bash
systemctl stop httpd

vi /etc/sysctl.conf

net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

:wq

sysctl -p
```
- sshd

```bash
# 確保 sshd 只監聽 ipv4，SSH 掛了就要靠 serial port 連進來了
vi /etc/ssh/sshd_config

AddressFamily inet

:wq

systemctl restart sshd
netstat -tpln
```

## Mail Service 安裝
- 完整 email address: john@example.com
- Mail Server `FQDN` : mail.example.com `替換後面[FQDN]的地方`
- Root Domain: example.com

```bash
yum -y install wget telnet git socat
yum -y remove sendmail # 刪除 sendmail
yum -y install mailx # 寄信測試工具
yum -y install postfix # SMTP
yum -y install dovecot # POP3
yum -y install cyrus-sasl-plain cyrus-sasl-md5 # SASL驗證
alternatives --config mta # 選擇 mta sendmail.postfix
```

### TLS 證書申請
- [acme.sh](https://github.com/Neilpang/acme.sh/)

```bash
cd ~
git clone https://github.com/acmesh-official/acme.sh.git
cd acme.sh
./acme.sh --install
# 重新登入 shell
acme.sh --version
acme.sh --set-default-ca --server letsencrypt
# 如果 server 上有開 web server
# 可把 --standalone 換成 --nginx or --apache
# 會自動修改 web config 驗證完再改回來 
acme.sh --issue -k 4096 --standalone --log -d [FQDN]
mkdir -p /etc/ssl/certs/[FQDN]
acme.sh --install-cert -d [FQDN] \
        --key-file /etc/ssl/certs/[FQDN]/private.key \
        --fullchain-file /etc/ssl/certs/[FQDN]/fullchain.cer

# 確認證書安裝成功，要有 fullchain.cer, private.key 兩個檔案
ls /etc/ssl/certs/[FQDN]

# 等整個 mail service 設定完再跑一次包含 --reloadcmd 的 --install-cert
# 這樣自動更新證書時就會順便重新載入 Mail Service
#acme.sh --install-cert -d [FQDN] \
#        --key-file /etc/ssl/certs/[FQDN]/private.key \
#        --fullchain-file /etc/ssl/certs/[FQDN]/fullchain.cer \
#        --reloadcmd "systemctl reload postfix dovecot"
```

### 建立一個實體使用者給 service 操作虛擬信箱
- username : vmail
- uid : 5000
- group name: vmail
- gid : 5000
- virtual domain base : /var/mail/vhosts

```bash
groupadd -g 5000 vmail
useradd -u 5000 -g vmail -s /sbin/nologin vmail
mkdir -p /var/mail/vhosts
chown vmail. /var/mail/vhosts
```

### postfix 設定

```bash
# 建立資料檔 後面設定會用到
touch /etc/postfix/virtual_domains
touch /etc/postfix/virtual_aliases \
    && echo -e "root@mail.example.com\troot@localhost" >> /etc/postfix/virtual_aliases \
    && postmap /etc/postfix/virtual_aliases
touch /etc/postfix/virtual_mailbox \
    && postmap /etc/postfix/virtual_mailbox

# DH parameters
openssl dhparam -out /etc/postfix/dh512.pem 512
openssl dhparam -out /etc/postfix/dh2048.pem 2048

# smtp auth 帳密檔
# 下面是 mailgun 設定
touch /etc/postfix/sasl_passwd
echo [smtp.mailgun.org]:2525 [YOUR_SMTP_USRNAME]:[YOUR_SMTP_PASSWORD] >> /etc/postfix/sasl_passwd
postmap /etc/postfix/sasl_passwd
```

```bash
vi /etc/postfix/main.cf

### 以下需搜尋各自變數的位置
default_privs = vmail
myhostname = mail.example.com
mydomain = example.com
myorigin = $myhostname
inet_interfaces = all
inet_protocols = ipv4
mydestination = $myhostname, localhost.$mydomain, localhost
mynetworks = 127.0.0.0/8
home_mailbox = Maildir/
smtpd_banner = SMTP ready.

# GCE 封鎖 port 25 , 必須使用外部SMTP且要用 25 以外的port
# 下面是 mailgun 設定 port 2525
relayhost = [smtp.mailgun.org]:2525

### 以下用 append 方式寫入

# 每封信不得超過30MB
message_size_limit = 31457280

# 每人信箱總量不得超過400MB, 0 表示無限
# mailbox_size_limit = 419430400
mailbox_size_limit = 0

# 同時可以收發總共多少封郵件, default : 100
# process限制, 機器爛可以設小一點
# default_process_limit = 10

# 每封郵件的標頭大小不得超過 4000 KB
header_size_limit = 4096000

# 防vrfy探測
disable_vrfy_command = yes

# ssl/tls
smtp_tls_CAfile = /etc/ssl/certs/ca-bundle.crt
smtp_tls_loglevel = 1
smtp_tls_security_level = encrypt
smtpd_use_tls = yes
smtpd_tls_cert_file = /etc/ssl/certs/[FQDN]/fullchain.cer
smtpd_tls_key_file = /etc/ssl/certs/[FQDN]/private.key
smtpd_tls_loglevel = 1
smtpd_tls_session_cache_database = btree:/var/lib/postfix/smtpd_scache
smtpd_tls_session_cache_timeout = 3600s
# Postfix >= 3.6 此參數將被忽略
smtpd_tls_dh512_param_file = /etc/postfix/dh512.pem
# Postfix >= 3.7 + OpenSSL >= 3.0.0 建議為空
# Postfix <= 3.7 建議使用 2048 位元
smtpd_tls_dh1024_param_file = /etc/postfix/dh2048.pem
smtp_tls_protocols = >=TLSv1
smtp_tls_mandatory_protocols = >=TLSv1
smtpd_tls_protocols = >=TLSv1
smtpd_tls_mandatory_protocols = >=TLSv1
tls_preempt_cipherlist = yes
smtp_tls_ciphers = high
smtpd_tls_ciphers = high

# smtp auth
broken_sasl_auth_clients = yes
smtp_sasl_security_options = noanonymous
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtpd_helo_required = yes
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain = $mydomain
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth

# smtpd antispam
### 參數對應
### smtpd_client_restrictions         ()
### smtpd_helo_restrictions           (HELO)
### smtpd_sender_restrictions         (MAIL FROM:)
### smtpd_recipient_restrictions      (RCPT TO:)
### smtpd_data_restrictions           (DATA)
### header_checks                     (DATA)
### body_checks                       (DATA)
smtpd_client_restrictions =
                permit_mynetworks,
                reject_non_fqdn_sender,
                reject_non_fqdn_recipient,
                permit

smtpd_helo_restrictions =
                permit_mynetworks,
                reject_invalid_helo_hostname,
                permit

smtpd_sender_restrictions =
                permit_mynetworks,
                reject_unknown_sender_domain,
                reject_non_fqdn_sender,
                permit

# permit_mynetworks, 允許來自 $mynetworks 的「寄信人」
# permit_sasl_authenticated, 允許經本機 SASL 驗證過的「寄信人」
# permit_auth_destination 只允許「收件人」在 Postfix 所管轄的網域
#   （由$mydestination定義），此選項避免你的伺服器變成跳板
smtpd_recipient_restrictions =
                permit_mynetworks,
                permit_sasl_authenticated,
                reject_unauth_destination,
                permit_auth_destination,
                reject

# virtual host
virtual_mailbox_base = /var/mail/vhosts
virtual_mailbox_domains = /etc/postfix/virtual_domains
virtual_mailbox_maps = hash:/etc/postfix/virtual_mailbox
virtual_alias_maps = hash:/etc/postfix/virtual_aliases
virtual_uid_maps = static:5000
virtual_gid_maps = static:5000
# 每人信箱總量不得超過400MB, 0 表示無限
# virtual_mailbox_limit = 419430400
virtual_mailbox_limit = 0

:wq
```

```bash
vi /etc/postfix/master.cf

smtp      inet  n       -       n       -       -       smtpd
# port 465
# postfix 舊版叫 smtps 新版叫 submissions
#smtps     inet  n       -       n       -       -       smtpd
submissions     inet  n       -       n       -       -       smtpd
  -o syslog_name=postfix/submissions
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
# port 587
submission inet n       -       n       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_auth_only=yes
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject

:wq
```

### dovecot 設定
```bash
vi /etc/dovecot/dovecot.conf

#protocols = imap pop3 lmtp
protocols = imap pop3

# listen = *, ::
listen = *

:wq
```

```bash
vi /etc/dovecot/conf.d/10-master.conf

service imap-login {
  # 關閉 imap
  inet_listener imap {
    #port = 143
    port = 0
  }
  inet_listener imaps {
    #port = 993
    #ssl = yes
  }
}
service pop3-login {
  # 關閉 pop3
  inet_listener pop3 {
    #port = 110
    port = 0
  }
  inet_listener pop3s {
    #port = 995
    #ssl = yes
  }
}

service auth {
  # Postfix smtp-auth
  # SMTP 登入使用 dovecot 的 passwd 驗證
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }
}

:wq
```

```bash
# 登入驗證
vi /etc/dovecot/conf.d/10-auth.conf

#disable_plaintext_auth = yes
disable_plaintext_auth = no

#auth_mechanisms = plain
auth_mechanisms = cram-md5 plain login

#!include auth-system.conf.ext # 加註這行
!include auth-passwdfile.conf.ext
!include auth-static.conf.ext

:wq
```

```bash
# 虛擬使用者密碼檔
touch /etc/dovecot/passwd
chown dovecot. /etc/dovecot/passwd

vi /etc/dovecot/conf.d/auth-passwdfile.conf.ext

# 清除其他設定，只留下面的

passdb {
  driver = passwd-file
  args = /etc/dovecot/passwd
}

:wq
```

```bash
# 虛擬網域 & 虛擬使用者 Home路徑
vi /etc/dovecot/conf.d/auth-static.conf.ext

# 清除其他設定，只留下面的

# %d = domain, %n = username
userdb {
  driver = static
  args = uid=vmail gid=vmail home=/var/mail/vhosts/%d/%n
}

:wq
```

```bash
# log 路徑 and 格式
mkdir /var/log/dovecot
vi /etc/dovecot/conf.d/10-logging.conf

log_path = /var/log/dovecot/dovecot.log
syslog_facility = local0
log_timestamp = "%Y-%m-%d %H:%M:%S "

:wq
```

```bash
# logrotate setting 每日 log 整理時 reload
# 每日任務 /etc/cron.daily/
# logrotate conf /etc/logrotate.conf
# logrotate include conf /etc/logrotate.d/
cat <<EOL > /etc/logrotate.d/dovecot
/var/log/dovecot/dovecot.log {
    sharedscripts
    postrotate
        /bin/systemctl reload dovecot.service > /dev/null 2>/dev/null || true
    endscript
}
EOL
```

```bash
# 使用者信箱資料夾
# Maildir 底下會自動建立 {cur,new,tmp} 三個資料夾
vi /etc/dovecot/conf.d/10-mail.conf

#mail_location =
mail_location = maildir:~/Maildir

:wq
```

```bash
# SSL證書設定
vi /etc/dovecot/conf.d/10-ssl.conf

ssl = required

#ssl_cert = </etc/pki/dovecot/certs/dovecot.pem
ssl_cert = </etc/ssl/certs/[FQDN]/fullchain.cer
#ssl_key = </etc/pki/dovecot/private/dovecot.pem
ssl_key = </etc/ssl/certs/[FQDN]/private.key

ssl_prefer_server_ciphers = yes

# 舊版
#ssl_dh_parameters_length = 2048
#ssl_protocols = !SSLv2 !SSLv3
#ssl_cipher_list = kEECDH:+kEECDH+SHA:kEDH:+kEDH+SHA:+kEDH+CAMELLIA:kECDH:+kECDH+SHA:kRSA:+kRSA+SHA:+kRSA+CAMELLIA:!aNULL:!eNULL:!SSLv2:!RC4:!MD5:!DES:!EXP:!SEED:!IDEA:!3DES
# 新版
ssl_min_protocol = TLSv1
ssl_cipher_list = PROFILE=SYSTEM

:wq
```

## Service 啟動
```bash
# config 檢查
postconf -n
postfix check

# 開機自動啟動
systemctl enable postfix
systemctl enable dovecot

# 立即啟動服務
systemctl start postfix
systemctl start dovecot

# 檢查 25、465、587、995、993 監聽
netstat -tpln
```

## 測試
- Shell Scripts 在文章最後面

```bash
# 新增網域
sh postfix_add_domain.sh "example.com"
# 新增使用者
sh postfix_add_user.sh "test@example.com" "mypassword"
```
- CRAM-MD5 登入

```bash
# smtps
openssl s_client -connect localhost:465
# or submission
# openssl s_client -starttls smtp -connect localhost:587

# 先送 helo 跟 AUTH
EHLO test
AUTH CRAM-MD5
# server 會回 334 跟著後面一大串的字是 [challenge] 複製起來 
# 下面的 [challenge] 是我亂打的 正常沒有那麼短
334 gg3be0+7788999 <- [challenge]
# 在另一個 terminal 用 shell 產出密碼
$ sh gen-cram-md5.sh "test@example.com" "mypassword" 'gg3be0+7788999'
dGVzdEBleGFtcGxlLmNvbSAoc3RkaW4pPSBhZTdlZjMxNDJlYTQ0ZTQzODI4YzdmODdhZWE1OWY5Ng==  # <= 這是密碼
# 複製 echo 出來的密碼 貼到 SMTP 登入的 terminal 送出
235 2.7.0 Authentication successful # 登入成功
```
- PLAIN 登入

```bash
# AUTH PLAIN 密碼 copy起來
printf "\0%s\0%s" "test@example.com" "mypassword" | base64
AHRlc3RAZXhhbXBsZS5jb20AbXlwYXNzd29yZA== # <= 這是密碼
# smtp 登入
telnet localhost 25
EHLO test
#AUTH PLAIN [密碼]
AUTH PLAIN AHRlc3RAZXhhbXBsZS5jb20AbXlwYXNzd29yZA==
235 2.7.0 Authentication successful # 登入成功
```
- 寄信測試

```bash
# 也可以測試寄到gmail之類的信箱
echo 'TEST MESSAGE BODY' | mail -s 'THIS IS DEMO SUBJECT' test@example.com
```

## Shell Scripts
### 新增虛擬網域
- postfix_add_domain.sh

```bash
#!/usr/bin/env bash

set -e

#
# postfix - add virtual domain
#
# Author: heatlai
#

# check inputs
if [ -z "$1" ]; then
    echo "usage:$0 [domain-name]"
    exit 128
fi
DOMAIN=$1

# 環境變數
VHOST=/var/mail/vhosts
VHOSTUSER=vmail
VHOSTGROUP=vmail

# 確認DOMAIN是否已存在
if grep -v "#" /etc/postfix/virtual_domains | grep -sq -e "^${DOMAIN}$"; then
    echo "error:'${DOMAIN}' is exists!"
else
    mkdir -p ${VHOST}/${DOMAIN}
    chown -R ${VHOSTUSER}.${VHOSTGROUP} ${VHOST}/${DOMAIN}
    echo "${DOMAIN}" >> /etc/postfix/virtual_domains
    systemctl reload postfix
fi
```
### 查看使用者密碼
- postfix_show_passwd.sh

```bash
#!/usr/bin/env bash

set -e

#
# postfix - show virtual user password
#
# Author: heatlai
#

# check inputs
if [ -z "$1" ]; then
    echo "usage:$0 [email]"
    exit 128
fi
EMAIL=$1
if ! echo "${EMAIL}" | grep -sq "@"; then
    echo "error:'${EMAIL}' is not email address!"
    exit 128
fi

if grep -sq "#${EMAIL}:" /etc/dovecot/passwd; then
    PASSWORD=$(grep "#${EMAIL}:" /etc/dovecot/passwd | cut -d':' -f2)
    echo "PLAIN Password: ${PASSWORD}"
else
    echo "error:'${EMAIL}' is not exists!"
    exit 1
fi
```
### 新增虛擬帳號
- postfix_add_user.sh

```bash
#!/usr/bin/env bash

set -e

#
# postfix - add virtual user
#
# Author: heatlai
#

# check inputs
if [ -z "$1" ]; then
    echo "usage:$0 [email] [password]"
    exit 128
fi

EMAIL=$1
if ! echo "${EMAIL}" | grep -sq "@"; then
    echo "error:'${EMAIL}' is not email address!"
    exit 128
fi

PASSWORD=$2
# check PASSWORD 沒輸入時自動產生密碼
if [ -z "${PASSWORD}" ]; then
    PASSWORD=$(openssl rand -base64 36 | tr -dc 'a-zA-Z0-9' | fold -w 16 | head -n 1)
fi

USERNAME=$(echo "${EMAIL}" | cut -d'@' -f1)
DOMAIN=$(echo "${EMAIL}" | cut -d'@' -f2)

if grep -v "#" /etc/dovecot/passwd | grep -sq "${EMAIL}"; then
    echo "error:'${EMAIL}'is exists!"
    exit 1
fi

if ! grep -v "#" /etc/postfix/virtual_domains | grep -sq -e "^${DOMAIN}$"; then
    echo "error:'${DOMAIN}' is not exists!"
    exit 1
fi

# 密碼加密
ENCRYPT_PASSWORD=$(doveadm pw -s CRAM-MD5 -p "${PASSWORD}")
# 寫入密碼
echo "${EMAIL}:${ENCRYPT_PASSWORD}" >> /etc/dovecot/passwd
# 保留明文密碼
echo "#${EMAIL}:${PASSWORD}" >> /etc/dovecot/passwd
# 寫入使用者信箱對應表
echo -e "${EMAIL}\t${DOMAIN}/${USERNAME}/Maildir/" >> /etc/postfix/virtual_mailbox
postmap /etc/postfix/virtual_mailbox
systemctl reload postfix
```
### 變更使用者密碼
- postfix_change_passwd.sh

```bash
#!/usr/bin/env bash

set -e

#
# postfix - change virtual user password
#
# Author: heatlai
#

# check inputs
if [ -z "$1" ]; then
    echo "usage:$0 [email] [password]"
    exit 128
fi

EMAIL=$1
if ! grep -v "#" /etc/dovecot/passwd | grep -sq "${EMAIL}"; then
    echo "error:'${EMAIL}' is not exists!"
    exit 1
fi

PASSWORD=$2
# check PASSWORD 沒輸入時自動產生密碼
if [ -z "${PASSWORD}" ]; then
    PASSWORD=$(openssl rand -base64 36 | tr -dc 'a-zA-Z0-9' | fold -w 16 | head -n 1)
fi

# 密碼加密
ENCRYPT_PASSWORD=$(doveadm pw -s CRAM-MD5 -p "${PASSWORD}")
# 刪除舊密碼
sed -i -e "/${EMAIL}/d" /etc/dovecot/passwd
# 寫入新密碼
echo "${EMAIL}:${ENCRYPT_PASSWORD}" >> /etc/dovecot/passwd
# 保留明文密碼
echo "#${EMAIL}:${PASSWORD}" >> /etc/dovecot/passwd
```

### 虛擬帳號列表
- postfix_user_list.sh

```bash
#!/usr/bin/env bash

set -e

#
# postfix - list virtual user
#
# Author: heatlai
#

grep -v "#" /etc/dovecot/passwd | cut -d':' -f1
```
### 刪除虛擬網域
- postfix_delete_domain.sh

```bash
#!/usr/bin/env bash

set -e

#
# postfix - delete virtual domain
#
# Author: heatlai
#

# check inputs
if [ -z "$1" ]; then
    echo "usage:$0 [domain-name]"
    exit 128
fi

DOMAIN=$1

VHOST=/var/mail/vhosts

if ! grep -v "#" /etc/postfix/virtual_domains | grep -sq -e "^${DOMAIN}$"; then
    echo "error:'${DOMAIN}' is not exists!"
    exit 1
fi

# 此網域的帳密刪除
sed -i -e "/@${DOMAIN}/d" /etc/dovecot/passwd
# 虛擬網域清單刪除
sed -i -e "/^${DOMAIN}/d" /etc/postfix/virtual_domains
# 虛擬帳號信箱對應表刪除
sed -i -e "/@${DOMAIN}/d" /etc/postfix/virtual_mailbox
# 信箱資料夾刪除
rm -rf ${VHOST}/${DOMAIN}
# 虛擬信箱別名刪除
sed -i -e "/@${DOMAIN}/d" /etc/postfix/virtual_aliases
# 套用新設定
postmap /etc/postfix/virtual_mailbox
postmap /etc/postfix/virtual_aliases
systemctl reload postfix
```
### 刪除虛擬帳號
- postfix_delete_user.sh

```bash
#!/usr/bin/env bash

set -e

#
# postfix - delete virtual user
#
# Author: heatlai
#

# check inputs
if [ -z "$1" ]; then
    echo "usage:$0 [email]"
    exit 128
fi

EMAIL=$1
if ! echo "${EMAIL}" | grep -sq "@"; then
    echo "error:'${EMAIL}' is not email address!"
    exit 128
fi
if ! grep -sq "${EMAIL}" /etc/dovecot/passwd; then
    echo "error:'${EMAIL}' is not exists!"
    exit 1
fi

USERNAME=$(echo "${EMAIL}" | cut -d'@' -f1)
DOMAIN=$(echo "${EMAIL}" | cut -d'@' -f2)

VHOST=/var/mail/vhosts

# 帳密刪除
sed -i -e "/${EMAIL}/d" /etc/dovecot/passwd
# 虛擬帳號信箱對應表刪除
sed -i -e "/${EMAIL}/d" /etc/postfix/virtual_mailbox
# 信箱資料夾刪除
rm -rf "${VHOST:?}/${DOMAIN:?}/${USERNAME:?}"
# 虛擬信箱別名刪除
sed -i -e "/${EMAIL}/d" /etc/postfix/virtual_aliases
# 套用新設定
postmap /etc/postfix/virtual_mailbox
postmap /etc/postfix/virtual_aliases
systemctl reload postfix
```

### SMTP AUTH CRAM-MD5 登入密碼產生器
- gen-cram-md5.sh

```bash
#!/usr/bin/env bash

# 
# SMTP AUTH CRAM-MD5 password generator
#
# Author: heatlai
#

if [ $# -lt 3 ] ; then
  echo "usage:$0 [email] [password] [challenge]"
  exit 128
fi

#
# like PHP hash_hmac()
#
function hash_hmac {
    local digest="$1"
    local data="$2"
    local key="$3"
    printf "${data}" | openssl dgst "-${digest}" -hmac "${key}" -binary | xxd -p
}

EMAIL=$1
PASSWORD=$2
CHALLENGE=$3

HASH_PASSWORD=$(hash_hmac 'md5' "$(printf "${CHALLENGE}" | base64 --decode)" "${PASSWORD}")

if [[ "$OSTYPE" == "darwin"* ]]; then
  # MacOS
  echo $(printf "${EMAIL} ${HASH_PASSWORD}" | base64)
else
  # Linux
  echo $(printf "${EMAIL} ${HASH_PASSWORD}" | base64 -w 0)
fi
```

## 其他設定
### 黑洞信箱 ( no-reply@example.com )
- 設定一個 address 名稱 `devnull` 儲存路徑指向 `/dev/null` 丟棄

```bash
vi /etc/aliases

# 增加下面這個
devnull:        /dev/null

:wq

# 套用 alias
newaliases
```
- 設定 `no-reply@example.com` 轉寄至 `devnull@localhost`

```bash
# 檔名來源 main.cf : 
# virtual_alias_maps = hash:/etc/postfix/virtual_aliases
vi /etc/postfix/virtual_aliases

no-reply@example.com       devnull

:wq
postmap /etc/postfix/virtual_aliases

# 重啟
systemctl reload postfix
```

### 系統郵件轉寄至私人信箱
- 設定 `root@localhost` 轉寄至 `john@example.com`

```bash
# 檔名來源 main.cf : 
# virtual_alias_maps = hash:/etc/postfix/virtual_aliases
vi /etc/postfix/virtual_aliases

root       john@example.com

:wq
postmap /etc/postfix/virtual_aliases

# 重啟
systemctl reload postfix
```

### SPF (Sender Policy Framework)
```bash
# SPF DNS Record
example.com. IN TXT "v=spf1 mx a ~all"
# Mailgun
example.com. IN TXT "v=spf1 include:mailgun.org ~all"
# Gmail
example.com. IN TXT "v=spf1 include:_spf.google.com ~all"

# 參數說明
# ip4:x.x.x.x 可指定 IP 或 CIDR x.x.x.x/24
# a:mail.example.com 不指定網域時就是指 example.com 本身
# mx:mail.example.com 不指定網域時就是指 example.com 本身
# include:mailgun.org 包含 mailgun.org 的 SPF 設定
# ~all SoftFail 錯誤但不拒收
# -all Fail 錯誤要拒收
```
### DKIM (DomainKeys Identified Mail) (自己發信時才需要)
#### 安裝設定
```bash
yum install -y opendkim

vim /etc/opendkim.conf

Mode sv
# 加註 KeyFile
#KeyFile        /etc/opendkim/keys/default.private
KeyTable /etc/opendkim/KeyTable
SigningTable refile:/etc/opendkim/SigningTable
ExternalIgnoreList refile:/etc/opendkim/TrustedHosts
InternalHosts refile:/etc/opendkim/TrustedHosts
```
#### Postfix 設定
```bash
vim /etc/postfix/main.cf

#
# opendkim
#
smtpd_milters = inet:localhost:8891
non_smtpd_milters = $smtpd_milters
milter_default_action = accept
```
#### DKIM 新增 Shell Script
```bash
#!/usr/bin/env bash

set -e

#
# DKIM - add domain key
#
# Author: heatlai
#

# check inputs
if [ -z "$1" ]; then
    echo "usage:$0 [domain]"
    exit 128
fi

if ! hash opendkim-genkey 2>/dev/null; then
    echo "command \"opendkim-genkey\" not exists. it's required."
    exit 1
fi

if ! hash opendkim 2>/dev/null; then
    echo "command \"opendkim\" not exists. it's required."
    exit 1
fi

if [ ! -f /etc/opendkim/keys/${DOMAIN}/default.private ]; then
    mkdir -p /etc/opendkim/keys/${DOMAIN}
    opendkim-genkey -D /etc/opendkim/keys/${DOMAIN}/ -d ${DOMAIN} -s default
    chown -R opendkim. /etc/opendkim/keys/${DOMAIN}
    echo "default._domainkey.${DOMAIN} ${DOMAIN}:default:/etc/opendkim/keys/${DOMAIN}/default.private" >> /etc/opendkim/KeyTable
    sed -i -e "/#\*@example.com/a\\*@${DOMAIN} default._domainkey.${DOMAIN}" /etc/opendkim/SigningTable
    opendkim reload
fi
```
- 執行後將 Public Key 加到 DNS Record

```bash
cat /etc/opendkim/keys/${DOMAIN}/default.txt
# 把 default.txt 的內容加到 DNS，像是下面這樣
# default._domainkey IN TXT "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDucWOlzIiyahWtQ9jgPjnfZp7x6K4L8Y20wFlmASr+eUlimC0w52SKIFancZILZeliPPjmaab4fHsvM1VJfF80EhMgTUKYQy7lOBBTjZVLmDfHAFzmUd3KOoaNbZ0tDiuayjrV1jAB82wHlrroGFBYQncpHbCUxlm31bu4WjLElwIDAQAB"
```

#### DKIM 刪除 Shell Script
- 執行後把 Public Key 從 DNS Record 上刪除

```bash
#!/usr/bin/env bash

set -e

#
# DKIM - delete domain key
#
# Author: heatlai
#

rm -rf "/etc/opendkim/keys/${DOMAIN}"
sed -i -e "/${DOMAIN}:/d" /etc/opendkim/KeyTable
sed -i -e "/@${DOMAIN} /d" /etc/opendkim/SigningTable
opendkim reload
```

### DMARC (Domain-based Message Authentication, Reporting & Conformance)
- 須先設定 SPF + DKIM

```bash
# DMARC DNS Record
_dmarc.example.com. IN TXT "v=DMARC1; p=none; rua=mailto:postmaster@example.com"
# rua 錯誤時送信給誰
```