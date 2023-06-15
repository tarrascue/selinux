# Часть 1
1.1 Добавление нестандартного порта в имеющийся тип
Проверяем статус сервиса nginx со стандартным портом 80

```
systemctl status nginx
  nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2023-06-15 08:22:59 UTC; 3s ago
  Process: 27098 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 27096 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 27094 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 27100 (nginx)
   CGroup: /system.slice/nginx.service
           ├─27100 nginx: master process /usr/sbin/nginx
           └─27101 nginx: worker process

Jun 15 08:22:59 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 15 08:22:59 selinux nginx[27096]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 15 08:22:59 selinux nginx[27096]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jun 15 08:22:59 selinux systemd[1]: Failed to read PID from file /run/nginx.pid: Invalid argument
Jun 15 08:22:59 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
Изменяем порт nginx 80->8090 и перезагружаем сервис:

```
systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
Смотрим перечень разрешенных портов для сервисов:

```
semanage port -l | grep http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
```
Используя утилиту semanage добавляем нестандартный порт 8090:
```
semanage port -a -t http_port_t -p tcp 8090

systemctl restart nginx
  nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2023-06-15 08:28:56 UTC; 5s ago
  Process: 27144 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 27143 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 27140 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 27147 (nginx)
   CGroup: /system.slice/nginx.service
           ├─27147 nginx: master process /usr/sbin/nginx
           └─27148 nginx: worker process

Jun 15 08:28:56 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 15 08:28:56 selinux nginx[27143]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 15 08:28:56 selinux nginx[27143]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jun 15 08:28:56 selinux systemd[1]: Failed to read PID from file /run/nginx.pid: Invalid argument
Jun 15 08:28:56 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Проверяем какой порт слушает nginx

```
ss -lntp | grep LISTEN | grep nginx
LISTEN     0      128          *:8090                     *:*			users:(("nginx",pid=27197,fd=6),("nginx",pid=27196,fd=6))
LISTEN     0      128         :::8090                    :::*			users:(("nginx",pid=27197,fd=7),("nginx",pid=27196,fd=7))
```

1.2 Формирование и установка модуля Selinux

Меняем порт nginx 8090->8095

```
systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```

Используя утилиту audit2why запустим анализ лога для формирования модуля

```
audit2allow -M httpd_add --debug < /var/log/audit/audit.log
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i httpd_add.pp
```

Инсталируем модуль
```
semodule -i httpd_add.pp

```

Проверяем

```
systemctl restart nginx

systemctl status nginx
  nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2023-06-15 08:32:38 UTC; 2s ago
  Process: 27194 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 27191 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 27190 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 27196 (nginx)
   CGroup: /system.slice/nginx.service
           ├─27196 nginx: master process /usr/sbin/nginx
           └─27197 nginx: worker process

Jun 15 08:32:38 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 15 08:32:38 selinux nginx[27191]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 15 08:32:38 selinux nginx[27191]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jun 15 08:32:38 selinux systemd[1]: Failed to read PID from file /run/nginx.pid: Invalid argument
Jun 15 08:32:38 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.

ss -lntp | grep LISTEN | grep nginx
LISTEN     0      128         :::8095                    :::*                   users:(("nginx",pid=27197,fd=7),("nginx",pid=27196,fd=7))
LISTEN     0      128         :::8090                    :::*			users:(("nginx",pid=27197,fd=7),("nginx",pid=27196,fd=7))
```

1.3 Использование переключателей setsebool

Меняем порт nginx 8095->8099

```
systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.

systemctl status nginx

 nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2023-06-15 08:33:08 UTC; 2s ago
  Process: 27194 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 27213 ExecStartPre=/usr/sbin/nginx -t 31m(code=exited, status=1/FAILURE)
  Process: 27211 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 27196 (code=exited, status=0/SUCCESS)

Jun 15 08:33:08 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Jun 15 08:33:08 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 15 08:33:08 selinux nginx[27213]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 15 08:33:08 selinux nginx[27213]: nginx: [emerg] bind() to 0.0.0.0:8099 failed (13: Permission denied)
Jun 15 08:33:08 selinux nginx[27213]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jun 15 08:33:08 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Jun 15 08:33:08 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server
Jun 15 08:33:08 selinux systemd[1]: Unit nginx.service entered failed state
Jun 15 08:33:08 selinux systemd[1]: nginx.service failed

```
Используя утилиту audit2why просматриваем лог
```
audit2why < /var/log/audit/audit.log

type=AVC msg=audit(1587026115.008:1025): avc:  denied  { name_bind } for  pid=27280 comm="nginx" src=8099 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:preupgrade_port_t:s0 tclass=tcp_socket permissive=0
	Was caused by:
	The boolean httpd_run_preupgrade was set incorrectly. 
	Description:
	Allow httpd to run preupgrade
	Allow access by executing:
	# setsebool -P httpd_run_preupgrade 1
```
Для устранения проблемы использовуем следующий переключатель: ```httpd_run_preupgrade``` установив в статус `1 (on)`__

```
setsebool -P httpd_run_preupgrade 1

systemctl restart nginx
systemctl status nginx
  nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2023-06-15 08:37:13 UTC; 4s ago
  Process: 27322 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 27320 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 27318 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 27324 (nginx)
   CGroup: /system.slice/nginx.service
           ├─27324 nginx: master process /usr/sbin/nginx
           └─27325 nginx: worker process

[root@otuslinux vagrant]# ss -lntp | grep LISTEN | grep nginx
LISTEN     0      128          *:8099                     *:*                   users:(("nginx",pid=27325,fd=6),("nginx",pid=27324,fd=6))
LISTEN     0      128         :::8099                    :::*                   users:(("nginx",pid=27325,fd=7),("nginx",pid=27324,fd=7))
```
В результате мы видим что все три механизма отрабатывают.

# Часть 2

После запуска стенда, при попытке внести изменения в зону ddns.lab, получаем ошибку

```
    nsupdate -k /etc/named.zonetransfer.key
    > server 192.168.50.10
    > zone ddns.lab
    > update add www.ddns.lab. 60 A 192.168.50.15
    > send
    update failed: SERVFAIL

```
Смотрим директорию, в которой находятся все конфиги зон

```
ls -Z /etc/named
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
ls -Z /etc/named/dynamic/
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab.view1
```
Использовав команды ls -Z /etc/named/* и ls -Z /etc/named/* видим что, причина не работоспособности - нет прав на изменения конфигурации для данного контекста.

Решение проблемы:

# Способ 1 - состояние permissive
Установить Selinux для named в состояние permissive

```
    semanage permissive -a named_t
```
Данная опция убирает блокирвоку но пробелена не решается.

# Способ 2 - контекст named_zone_t

данный контекст применяется для файлов зоны которые можно изменять

```
chcon -t named_zone_t /etc/named/dynamic/*
chcon -t named_zone_t /etc/named/*
```
Меняем контекст
```
ls -Z /etc/named/dynamic/
-rw-rw----. named named system_u:object_r:named_zone_t:s0 named.ddns.lab
-rw-rw----. named named system_u:object_r:named_zone_t:s0 named.ddns.lab.view1

ls -Z /etc/named/
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```
Результат положительный
```
    [vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
    > server 192.168.50.10
    > zone ddns.lab 
    > update add www.ddns.lab. 60 A 192.168.50.15
    > send
```
