---
layout: post
title:  '[CentOS] 安裝 Supervisor Daemon管理'
subtitle: 'CentOS - Install Supervisor'
background: '/img/posts/01.jpg'

date: 2018-10-18

tags: [CentOS]
keywords: [Supervisor, Install]
---

# 安裝
```
yum install python-setuptools
easy_install supervisor
echo_supervisord_conf > /etc/supervisord.conf
```

# 設定
#### 建立 "自訂 Daemon 設定檔" 資料夾
```
mkdir -p /etc/supervisor/conf.d
```
- supervisor 主設定檔路徑 : `/etc/supervisord.conf`
- 以下只寫出 `supervisord.conf` 常用的設定

```ini
[unix_http_server]
file=/var/run/supervisor.sock   ; the path to the socket file

; WEB 管理介面 (視需求決定是否要開)
;[inet_http_server]         ; inet (TCP) server disabled by default
;port=0.0.0.0:9001        ; ip_address:port specifier, *:port for all iface
;username=user              ; default is no username (open server)
;password=123               ; default is no password (open server)

[supervisord]
logfile=/var/log/supervisord.log ; main log file; default $CWD/supervisord.log
logfile_maxbytes=50MB        ; max main logfile bytes b4 rotation; default 50MB
logfile_backups=10           ; # of main logfile backups; 0 means none, default 10
loglevel=info                ; log level; default info; others: debug,warn,trace
pidfile=/var/run/supervisord.pid ; supervisord pidfile; default supervisord.pid
nodaemon=false               ; start in foreground if true; default false
minfds=1024                  ; min. avail startup file descriptors; default 1024
minprocs=200                 ; min. avail process descriptors;default 200

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl=unix:///var/run/supervisor.sock ; use a unix:// URL  for a unix socket
;serverurl=http://127.0.0.1:9001 ; use an http:// url to specify an inet socket
;username=chris              ; should be same as in [*_http_server] if set
;password=123                ; should be same as in [*_http_server] if set

[include]
; 自訂 Daemon 設定檔路徑
files = /etc/supervisor/conf.d/*.ini
```

#### 自訂 Daemon 設定
- 範例為 Ratchet (PHP的WebSocket套件)
```
vim /etc/supervisor/conf.d/ratchet.ini
```

```ini
[program:ratchet]
command                 = bash -c "ulimit -n 10000; exec /usr/bin/php /root/websocket-start.php" ; 要執行的CMD
;directory              = /root ; 先cd到directory再執行command (非必要, 全部寫絕對路徑也可以)
;process_name=%(program_name)s_%(process_num)02d
process_name            = Ratchet ; process 名稱
numprocs                = 1 ; process 數量
autostart               = true ; 自動啟動
autorestart             = true ; 自動重啟
user                    = root ; 執行的User帳號
stdout_logfile          = /var/log/supervisor/ratchet.log ; output log
stdout_logfile_maxbytes = 1MB ; log file size, 超過會自動循環
stdout_logfile_backups  = 5
stdout_capture_maxbytes = 1MB
redirect_stderr         = true ; error_log 直接寫到 stdout_logfile
;stderr_logfile          = /var/log/supervisor/ratchet_error.log
;stderr_logfile_maxbytes = 1MB
```

# 啟動 service
```
sudo service supervisord start
sudo supervisorctl status | awk '{print $1, $2}'
```

# 開機自動啟動 supervisor (CentOS 6)
#### Startup Script
```
sudo vi /etc/rc.d/init.d/supervisord
```
#### /etc/rc.d/init.d/supervisord 內容
```bash
#!/bin/bash
#
# supervisord   Startup script for the Supervisor process control system
#
# chkconfig: 345 83 04
# description: Supervisor is a client/server system that allows \
#   its users to monitor and control a number of processes on \
#   UNIX-like operating systems.
# processname: supervisord
# config: /etc/supervisord.conf
# config: /etc/sysconfig/supervisord
# pidfile: /var/run/supervisord.pid
#
### BEGIN INIT INFO
# Provides: supervisord
# Required-Start: $all
# Required-Stop: $all
# Short-Description: start and stop Supervisor process control system
# Description: Supervisor is a client/server system that allows
#   its users to monitor and control a number of processes on
#   UNIX-like operating systems.
### END INIT INFO

# Source function library
. /etc/rc.d/init.d/functions

# Source system settings
if [ -f /etc/sysconfig/supervisord ]; then
    . /etc/sysconfig/supervisord
fi

# Path to the supervisorctl script, server binary,
# and short-form for messages.
supervisorctl=${SUPERVISORCTL-/usr/local/bin/supervisorctl}
supervisord=${SUPERVISORD-/usr/local/bin/supervisord}
prog=supervisord
pidfile=${PIDFILE-/var/run/supervisord.pid}
lockfile=${LOCKFILE-/var/lock/subsys/supervisord}
sockfile=${SOCKFILE-/var/run/supervisord.sock}
STOP_TIMEOUT=${STOP_TIMEOUT-60}
OPTIONS="${OPTIONS--c /etc/supervisord.conf}"
RETVAL=0

start() {
    echo -n $"Starting $prog: "
    daemon --pidfile=${pidfile} $supervisord $OPTIONS
    RETVAL=$?
    echo
    if [ $RETVAL -eq 0 ]; then
        touch ${lockfile}
        $supervisorctl $OPTIONS status
    fi
    return $RETVAL
}

stop() {
    echo -n $"Stopping $prog: "
    killproc -p ${pidfile} -d ${STOP_TIMEOUT} $supervisord
    RETVAL=$?
    echo
    [ $RETVAL -eq 0 ] && rm -rf ${lockfile} ${pidfile} ${sockfile}
}

reload() {
    echo -n $"Reloading $prog: "
    LSB=1 killproc -p $pidfile $supervisord -HUP
    RETVAL=$?
    echo
    if [ $RETVAL -eq 7 ]; then
        failure $"$prog reload"
    else
        $supervisorctl $OPTIONS status
    fi
}

restart() {
    stop
    start
}

case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    status)
        status -p ${pidfile} $supervisord
        RETVAL=$?
        [ $RETVAL -eq 0 ] && $supervisorctl $OPTIONS status
        ;;
    restart)
        restart
        ;;
    condrestart|try-restart)
        if status -p ${pidfile} $supervisord >&/dev/null; then
          stop
          start
        fi
        ;;
    force-reload|reload)
        reload
        ;;
    *)
        echo $"Usage: $prog {start|stop|restart|condrestart|try-restart|force-reload|reload}"
        RETVAL=2
esac

exit $RETVAL
```
#### 加入到 chkconfig
```
sudo chmod 755 /etc/rc.d/init.d/supervisord
sudo chkconfig --add supervisord
sudo chkconfig supervisord on # 非必要, supervisord 已寫 runlevel:345
```

# 開機自動啟動 supervisor (CentOS 7)
#### Startup Script
```
sudo vi /etc/systemd/system/supervisord.service
```
#### /etc/systemd/system/supervisord.service 內容
```bash
[Unit]
Description=Supervisor process control system for UNIX
Documentation=http://supervisord.org
After=network.target

[Service]
ExecStart=/usr/bin/supervisord -n -c /etc/supervisord.conf
ExecStop=/usr/bin/supervisorctl $OPTIONS shutdown
ExecReload=/usr/bin/supervisorctl $OPTIONS reload
KillMode=process
Restart=on-failure
RestartSec=50s

[Install]
WantedBy=multi-user.target
```
#### 確認 service 設定
```
systemctl list-unit-files --type=service
systemctl start supervisord
systemctl status supervisord
systemctl stop supervisord
systemctl enable supervisord.service
```