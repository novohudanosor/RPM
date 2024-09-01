# RPM
## Управление пакетами. Дистрибьюция софта.

## Цель:
* создать свой RPM;
* создать свой репозиторий с RPM;

1. Сборка пакета NGINX с поддержкой openssl
Установка ПО необходимого для сборки пакетов и загрузка исходных кодов программ производится во время запуска образа операционной системы.
1. Установка SRPM пакета NGINX, после которой создаётся дерево каталогов для сборки rpmbuild:
```
[root@rpmtest ~]# rpm -i nginx-1.20.2-1.el7.ngx.src.rpm 
warning: nginx-1.20.2-1.el7.ngx.src.rpm: Header V4 RSA/SHA1 Signature, key ID 7bd9bf62: NOKEY
[root@rpmtest ~]# ll
total 17832
-rw-r--r--.  1 root root 11924330 Sep  9 16:43 OpenSSL_1_1_1-stable.zip
-rw-------.  1 root root     5207 Dec  4  2020 anaconda-ks.cfg
-rw-r--r--.  1 root root  1082461 Sep  9 16:43 nginx-1.20.2-1.el7.ngx.src.rpm
drwxr-xr-x. 19 root root     4096 Sep 11  2023 openssl-OpenSSL_1_1_1-stable
-rw-------.  1 root root     5006 Dec  4  2020 original-ks.cfg
-rw-r--r--.  1 root root  5222976 Sep  9 16:43 percona-orchestrator-3.2.6-2.el8.x86_64.rpm
drwxr-xr-x.  4 root root       34 Sep  9 16:44 rpmbuild
```
2. Разрешение зависимостей пакета NGINX:
```
[root@rpmtest ~]# yum-builddep rpmbuild/SPECS/nginx.spec
...
```
3. Редактируем spec-файл NGINX для того, что бы соответствующий пакет собирался с поддержкой SSL. Для этого необходимо в секцию ./configure файла nginx.spec, добавить опцию сборки --with-openssl=/root/openssl-OpenSSL_1_1_1-stable, в которой содержится путь к исходным
кодам OpenSSL.
4. Сборка RPM пакета NGINX:
```
[root@rpmtest ~]# rpmbuild -bb rpmbuild/SPECS/nginx.spec
...
[root@rpmtest ~]# ll rpmbuild/RPMS/x86_64/
total 4864
-rw-r--r--. 1 root root 2444220 Sep  9 17:11 nginx-1.20.2-1.el8.ngx.x86_64.rpm
-rw-r--r--. 1 root root 2533336 Sep  9 17:11 nginx-debuginfo-1.20.2-1.el8.ngx.x86_64.rpm
```
5. Установка NGINX:
```
[root@rpmtest ~]# yum localinstall -y rpmbuild/RPMS/x86_64/nginx-1.20.2-1.el8.ngx.x86_64.rpm
...
Installed:
  nginx-1:1.20.2-1.el8.ngx.x86_64                                                                                                        

Complete!
```
6. Запускаем NGINX и проверка статуса:
```
[root@rpmtest ~]# systemctl start nginx
[root@rpmtest ~]# systemctl status nginx
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2024-09-01 17:18:15 UTC; 14s ago
     Docs: http://nginx.org/en/docs/
  Process: 46855 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=0/SUCCESS)
 Main PID: 46856 (nginx)
    Tasks: 5 (limit: 12419)
   Memory: 5.1M
   CGroup: /system.slice/nginx.service
           ├─46856 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
           ├─46857 nginx: worker process
           ├─46858 nginx: worker process
           ├─46859 nginx: worker process
           └─46860 nginx: worker process

Sep 9 17:18:15 rpmtest systemd[1]: Starting nginx - high performance web server...
Sep 9 17:18:15 rpmtest systemd[1]: Started nginx - high performance web server.
```
7. Создание локального репозитория
```
[root@rpmtest ~]# mkdir /usr/share/nginx/html/repo
[root@rpmtest ~]# cp rpmbuild/RPMS/x86_64/nginx-1.20.2-1.el8.ngx.x86_64.rpm /usr/share/nginx/html/repo
[root@rpmtest ~]# cp percona-orchestrator-3.2.6-2.el8.x86_64.rpm /usr/share/nginx/html/repo
[root@rpmtest ~]# ll /usr/share/nginx/html/repo
total 7492
-rw-r--r--. 1 root root 2444220 Sep 9 17:22 nginx-1.20.2-1.el8.ngx.x86_64.rpm
-rw-r--r--. 1 root root 5222976 Sep 9 17:22 percona-orchestrator-3.2.6-2.el8.x86_64.rpm
```
8. Инициализация локального репозитория:
```
[root@rpmtest ~]# createrepo /usr/share/nginx/html/repo/
Directory walk started
Directory walk done - 2 packages
Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished
```
9. Настроим NGINX для доступа к листингу каталога репозитория. В location / файла ``` /etc/nginx/conf.d/default.conf ``` необходимо добавить дерективу **autoindex on**. После чего проверить корректность конфигурационного файла и перезапустить сервис NGINX.
```
[root@rpmtest ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@rpmtest ~]# nginx -s reload

```
10.  Проверка работы веб-страницы командой curl
```
[root@rpmtest ~]# curl -a http://localhost/repo/
<html>
<head><title>Index of /repo/</title></head>
<body>
<h1>Index of /repo/</h1><hr><pre><a href="../">../</a>
<a href="repodata/">repodata/</a>                                          1-Sep-2024 17:26                   -
<a href="nginx-1.20.2-1.el8.ngx.x86_64.rpm">nginx-1.20.2-1.el8.ngx.x86_64.rpm</a>                  1-Sep-2024 17:22             2444220
<a href="percona-orchestrator-3.2.6-2.el8.x86_64.rpm">percona-orchestrator-3.2.6-2.el8.x86_64.rpm</a>        1-Sep-2024 17:22             5222976
</pre><hr></body>
</html>
```
11. Добавление репозитория в ```/etc/yum.repos.d/otus.repo```
```
[root@rpmtest ~]# cat >> /etc/yum.repos.d/otus.repo << EOF
[otus]
name=otus-linux
baseurl=http://localhost/repo
gpgcheck=0
enabled=1
EOF
```
12. Проверка подключения репозитория и наличия содержимого:
```
[root@rpmtest ~]# yum repolist enabled | grep otus
Failed to set locale, defaulting to C.UTF-8
otus                            otus-linux
[root@rpmtest ~]# yum list | grep otus
Failed to set locale, defaulting to C.UTF-8
otus-linux                                      1.3 MB/s | 2.8 kB     00:00    
percona-orchestrator.x86_64                            2:3.2.6-2.el8                                          otus        
```
* Название пакета NGINX, из локального репозитория не выводится, так как эта информация перекрывается данными репозиториев CentOS.
13. Устанавливаем пакет из локального репозитория
```
[root@rpmtest ~]# yum install -y percona-orchestrator.x86_64
...
Installed:
  jq-1.5-12.el8.x86_64          oniguruma-6.8.2-2.el8.x86_64          percona-orchestrator-2:3.2.6-2.el8.x86_64         

Complete!
```




