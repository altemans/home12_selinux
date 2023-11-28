# Home 12 selinux

<details>
  <summary>1 часть</summary>

<details>
  <summary>запуск  vagrant</summary>
 
```
сразу добавим установку policycoreutils-python в вагрант файл

altemans@Home01:~/otus/home12_selinux$ vagrant up
Bringing machine 'selinux' up with 'virtualbox' provider...
==> selinux: Importing base box 'centos/7'...
==> selinux: Matching MAC address for NAT networking...
==> selinux: Setting the name of the VM: home12_selinux_selinux_1701176289296_87622
==> selinux: Clearing any previously set network interfaces...
==> selinux: Preparing network interfaces based on configuration...
    selinux: Adapter 1: nat
==> selinux: Forwarding ports...
    selinux: 4881 (guest) => 4881 (host) (adapter 1)
    selinux: 22 (guest) => 2222 (host) (adapter 1)
==> selinux: Running 'pre-boot' VM customizations...

...

    selinux: ● nginx.service - The nginx HTTP and reverse proxy server
    selinux:    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
    selinux:    Active: failed (Result: exit-code) since Tue 2023-11-28 13:01:14 UTC; 30ms ago
    selinux:   Process: 2904 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
    selinux:   Process: 2903 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    selinux: 
    selinux: Nov 28 13:01:14 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
    selinux: Nov 28 13:01:14 selinux nginx[2904]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    selinux: Nov 28 13:01:14 selinux nginx[2904]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
    selinux: Nov 28 13:01:14 selinux nginx[2904]: nginx: configuration file /etc/nginx/nginx.conf test failed
    selinux: Nov 28 13:01:14 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
    selinux: Nov 28 13:01:14 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
    selinux: Nov 28 13:01:14 selinux systemd[1]: Unit nginx.service entered failed state.
    selinux: Nov 28 13:01:14 selinux systemd[1]: nginx.service failed.
The SSH command responded with a non-zero exit status. Vagrant
assumes that this means the command failed. The output for this command
should be in the log above. Please read the output to determine what
went wrong.
```  
При старте наблюдаем ошибку запуска  nginx на порту 4881 из-за отсутствия прав  

заходим на виртуалку  

</details>



<details>
  <summary>vagrant ssh</summary>
 
 проверяем запущен ли файрвол, конфигурацию nginx состояние selinux   
 
 ```
[root@selinux ~]# systemctl status firewalld.service 
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
 ```

выключен

```
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

все ок

```
[root@selinux ~]# getenforce
Enforcing
```
selinux работает в активном режиме

</details>

<details>
  <summary>Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool</summary>

ищем сообщения о блокировке в логе  
```
[root@selinux ~]# cat /var/log/audit/audit.log | grep denied
type=AVC msg=audit(1701176474.108:835): avc:  denied  { name_bind } for  pid=2904 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```

```
[root@selinux ~]# grep 1701176474.108:835 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1701176474.108:835): avc:  denied  { name_bind } for  pid=2904 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly. 
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
```

утилита рекомендует разрешить параметр nis_enabled,  

```
[root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2023-11-28 13:29:34 UTC; 6s ago
  Process: 3102 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3100 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3098 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3104 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3104 nginx: master process /usr/sbin/nginx
           └─3106 nginx: worker process

Nov 28 13:29:34 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 28 13:29:34 selinux nginx[3100]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 28 13:29:34 selinux nginx[3100]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Nov 28 13:29:34 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

демон запустился, проверяем  

```
root@selinux ~]curl http://127.0.0.1:4881
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
  <style rel="stylesheet" type="text/css"> 

        html {
        background-image:url(img/html-background.png);
        background-color: white;
        font-family: "DejaVu Sans", "Liberation Sans", sans-serif;
        font-size: 0.85em;
```

отключаем   
setsebool -P nis_enabled off


</details>


<details>
  <summary>разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип</summary>

Поиск имеющегося типа, для http трафика: semanage port -l | grep http

```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

добавляем порт, в методичке ошибка, пропущена буква s  
semanage port -a -t http_port_t -p tcp 4881  
проверяем  

```
[root@selinux ~]# systemctl restart nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2023-11-28 13:51:00 UTC; 5s ago
  Process: 21934 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21932 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21930 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21936 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21936 nginx: master process /usr/sbin/nginx
           └─21938 nginx: worker process

Nov 28 13:51:00 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 28 13:51:00 selinux nginx[21932]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 28 13:51:00 selinux nginx[21932]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Nov 28 13:51:00 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

возвращаем как было  
semanage port -d -t http_port_t -p tcp 4881

```
[root@selinux ~]# semanage port -l | grep  http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux ~]# 
```

</details>

<details>
  <summary>Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux</summary>

Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту: 
grep nginx /var/log/audit/audit.log | audit2allow -M nginx

```
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp

[root@selinux ~]# 
```

применим сформированный модуль

```
semodule -i nginx.pp


[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2023-11-28 13:56:53 UTC; 9s ago
  Process: 21985 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21982 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21981 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21987 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21987 nginx: master process /usr/sbin/nginx
           └─21989 nginx: worker process

Nov 28 13:56:53 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 28 13:56:53 selinux nginx[21982]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 28 13:56:53 selinux nginx[21982]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Nov 28 13:56:53 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
[root@selinux ~]# 
```

удаляем модуль

```
[root@selinux ~]# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
```

</details>
</details>

<details>
  <summary>2 часть</summary>




<details>
  <summary>клонируем репу</summary>
```
altemans@Home01:~/otus/home12_selinux/2$ git clone https://github.com/mbfx/otus-linux-adm.git
Клонирование в «otus-linux-adm»...
remote: Enumerating objects: 558, done.
remote: Counting objects: 100% (456/456), done.
remote: Compressing objects: 100% (303/303), done.
remote: Total 558 (delta 125), reused 396 (delta 74), pack-reused 102
Получение объектов: 100% (558/558), 1.38 МиБ | 556.00 КиБ/с, готово.
Определение изменений: 100% (140/140), готово.
```
</details>

<details>
  <summary>vagrant up</summary>

```
altemans@Home01:~/otus/home12_selinux/2/otus-linux-adm/selinux_dns_problems$ vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

```
</details>

<details>
  <summary>vagrant ssh client</summary>

Попробуем внести изменения в зону: nsupdate -k /etc/named.zonetransfer.key

```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
[vagrant@client ~]$ 
```
получаем ошибку  
cat /var/log/audit/audit.log | audit2why  
на клиенте ошибок не найдено  
  
идем на сервер  


</details>



<details>
  <summary>vagrant ssh ns01 </summary>

ищем ошибку на сервере

```
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1701184701.452:1894): avc:  denied  { write } for  pid=5167 comm="isc-worker0000" name="named" dev="sda1" ino=67541567 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:named_zone_t:s0 tclass=dir permissive=0

        Was caused by:
        The boolean named_write_master_zones was set incorrectly. 
        Description:
        Allow named to write master zones

        Allow access by executing:
        # setsebool -P named_write_master_zones 1
type=AVC msg=audit(1701185086.017:1919): avc:  denied  { create } for  pid=5167 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

[root@ns01 ~]# 
```
проверяем контекст на каталоге

```
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
[root@ns01 ~]# sudo semanage fcontext -l | grep named
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0 
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0 
/etc/unbound(/.*)?                                 all files          system_u:object_r:named_conf_t:s0 
```
правим контекст

```
[root@ns01 ~]# sudo chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```
 **Помимо измененного контекста требуется разрешить BIND записывать файлы главной зоны для динамического DNS или передачи зон.**  
 **Иначе получаем ошибку**

```
type=AVC msg=audit(1701189805.383:2003): avc:  denied  { write } for  pid=5167 comm="isc-worker0000" name="dynamic" dev="sda1" ino=5385623 scontext=system_u:system_r:named_t:s0 tcontext=unconfined_u:object_r:named_zone_t:s0 tclass=dir permissive=0

        Was caused by:
        The boolean named_write_master_zones was set incorrectly. 
        Description:
        Allow named to write master zones

        Allow access by executing:
        # setsebool -P named_write_master_zones 1
```

 влючаем  
```
setsebool -P named_write_master_zones 1
```

</details>


<details>
  <summary>проверяем</summary>

```
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 13774
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 15 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Tue Nov 28 16:58:12 UTC 2023
;; MSG SIZE  rcvd: 96

```

</details>


</details>