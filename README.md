# Практика с SELinux #
1. Запустить NGINX на нестандартном порту тремя разными способами:
   - переключатели setsebool;
   - добавление нестандартного порта в имеющийся тип;
   - формирование и установка модуля SELinux.
### Исходные данные ###
&ensp;&ensp;ПК на Linux c 8 ГБ ОЗУ или виртуальная машина (ВМ) с включенной Nested Virtualization.<br/>
&ensp;&ensp;Предварительно установленное и настроенное ПО:<br/>
&ensp;&ensp;&ensp;Hashicorp Vagrant (https://www.vagrantup.com/downloads);<br/>
&ensp;&ensp;&ensp;Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads).<br/>
&ensp;&ensp;&ensp;Ansible (https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).<br/>
&ensp;&ensp;&ensp;Все действия проводились с использованием Vagrant 2.4.0, VirtualBox 7.0.14,<br/> Ansible 9.3.0 и образа CentOS 7 версии 1804_2.<br/> 
### Ход решения ###
#### Разрешение в SELinux работы сервиса NGINX на порту TCP 4881 с помощью переключателей setsebool ####
1. С помощью предложенного Vagrant-файла запускаем виртуальную машину с установленным NGINX,<br/>
который работает на порту TCP 4881. Этот порт проброшен до хостовой машины. SELinux включен.<br/>
2. Во время развёртывания стенда, попытка запуска NGINX завершается с ошибкой. Проверяем состояние <br/>
сервиса NGINX и пробуем повторно его запустить:<br/>
```shell
[vagrant@selinux ~]$ sudo -i
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
[root@selinux ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx
 nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2024-05-30 12:27:19 UTC; 10s ago
  Process: 2242 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 2240 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)

May 30 12:27:19 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 30 12:27:19 selinux nginx[2242]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 30 12:27:19 selinux nginx[2242]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
May 30 12:27:19 selinux nginx[2242]: nginx: configuration file /etc/nginx/nginx.conf test failed
May 30 12:27:19 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
May 30 12:27:19 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
May 30 12:27:19 selinux systemd[1]: Unit nginx.service entered failed state.
May 30 12:27:19 selinux systemd[1]: nginx.service failed.
```
3. Проверка корректности конфигурационного файла NGINX и режима работы системы SELinux:<br/>
```shell
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

[root@selinux ~]# getenforce
Enforcing
```
4. С помощью утилиты audit2why производим оценку audit-лога /var/log/audit/audit.log:<br/>
```shell
[root@selinux ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1717072185.958:557): avc:  denied  { name_bind } for  pid=2288 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```
&ensp;&ensp;В ходе анализа содержимого лога установлено, что запуск сервиса NGINX на нестандартном порту<br/>
блокируется системой SELinux. Кроме того, утилита audit2why сообщает, что для разрешения проблемы необходимо<br/>
установить значение параметра nis_enabled в 1.<br/>

5. Установка параметра nis_enabled в 1, запуск, проверка состояния и работоспособности сервиса NGINX:<br/>
```shell
[root@selinux ~]# setsebool -P nis_enabled on

[root@selinux ~]# systemctl start nginx

[root@selinux ~]# systemctl status nginx
 nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2024-05-30 12:32:51 UTC; 12s ago
  Process: 2337 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2335 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2334 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2339 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2339 nginx: master process /usr/sbin/nginx
           └─2341 nginx: worker process

May 30 12:32:51 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 30 12:32:51 selinux nginx[2335]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 30 12:32:51 selinux nginx[2335]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 30 12:32:51 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.

[root@selinux ~]# ss -ntlp
State      Recv-Q Send-Q                                                                                             Local Address:Port                                                                                                            Peer Address:Port              
LISTEN     0      128                                                                                                            *:111                                                                                                                        *:*                   users:(("rpcbind",pid=576,fd=8))
LISTEN     0      128                                                                                                            *:4881                                                                                                                       *:*                   users:(("nginx",pid=2341,fd=6),("nginx",pid=2339,fd=6))
LISTEN     0      128                                                                                                            *:22                                                                                                                         *:*                   users:(("sshd",pid=818,fd=3))
LISTEN     0      100                                                                                                    127.0.0.1:25                                                                                                                         *:*                   users:(("master",pid=1020,fd=13))
LISTEN     0      128                                                                                                           :::111                                                                                                                       :::*                   users:(("rpcbind",pid=576,fd=11))
LISTEN     0      128                                                                                                           :::4881                                                                                                                      :::*                   users:(("nginx",pid=2341,fd=7),("nginx",pid=2339,fd=7))
LISTEN     0      128                                                                                                           :::22                                                                                                                        :::*                   users:(("sshd",pid=818,fd=4))
LISTEN     0      100                                                                                                          ::1:25                                                                                                                        :::*                   users:(("master",pid=1020,fd=14))

[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
```
![изображение](https://github.com/DemBeshtau/12_1_DZ/assets/149678567/7ab46b3d-90b0-41dc-9a6d-02099d612a1f)

6. Возврат настроек к прежнему состоянию:<br/>
```shell
[root@selinux ~]# getsebool -P nis_enabled off

[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
#### Разрешение в SELinux работы сервиса NGINX на порту TCP 4881<br/> с помощью добавления нестандартного порта в имеющийся тип ####
7. Просмотр имеющегося типа для http трафика:
```shell
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_http_port_t            tcp      5989
```
&ensp;&ensp;Видно, что порт TCP 4881 в числе разрешённых отсутствует.

8. Добавление порта TCP 4881  в тип http_port_t, запуск, проверка состояния и работоспособности сервиса NGINX:<br/>
```shell
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881 

[root@selinux ~]# systemctl start nginx

[root@selinux ~]# systemctl status nginx
 nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2024-05-30 12:40:14 UTC; 18s ago
  Process: 2415 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2413 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2412 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2417 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2417 nginx: master process /usr/sbin/nginx
           └─2418 nginx: worker process

May 30 12:40:14 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 30 12:40:14 selinux nginx[2413]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 30 12:40:14 selinux nginx[2413]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 30 12:40:14 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.

[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_http_port_t            tcp      5989
```
![изображение](https://github.com/DemBeshtau/12_1_DZ/assets/149678567/93405fc0-963f-428f-bd81-c7ef2211e1c1)

9. Возврат настроек к прежнему состоянию - удаление нестандартного порта из имеющегося типа:<br/>
```shell
[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881

[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_http_port_t            tcp      5989

[root@selinux ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
#### Разрешение в SELinux работы сервиса NGINX на порту TCP 4881 с помощью формирования и установки модуля ####
10. Подготовка модуля на основе audit-лога, разрешающего работу сервиса NGINX на нестандартном порту. Модуль формирует утилита audit2allow:
```shell
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx 
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp

[root@selinux ~]# semodule -i nginx.pp
[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
 nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2024-05-30 12:44:59 UTC; 15s ago
  Process: 2492 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2490 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 2489 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2494 (nginx)
   CGroup: /system.slice/nginx.service
           ├─2494 nginx: master process /usr/sbin/nginx
           └─2496 nginx: worker process

May 30 12:44:59 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 30 12:44:59 selinux nginx[2490]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 30 12:44:59 selinux nginx[2490]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 30 12:44:59 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
![изображение](https://github.com/DemBeshtau/12_1_DZ/assets/149678567/22d8ba68-741a-46df-bd4e-53c337ad9380)
11. Возврат настроек к прежнему состоянию - удаление модуля SELinux:<br/>
```shell
[root@selinux ~]# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).

[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.

[root@selinux ~]# systemctl status nginx
 nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Thu 2024-05-30 12:47:39 UTC; 19s ago
  Process: 2492 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 2517 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 2516 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 2494 (code=exited, status=0/SUCCESS)

May 30 12:47:39 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
May 30 12:47:39 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 30 12:47:39 selinux nginx[2517]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 30 12:47:39 selinux nginx[2517]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
May 30 12:47:39 selinux nginx[2517]: nginx: configuration file /etc/nginx/nginx.conf test failed
May 30 12:47:39 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
May 30 12:47:39 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
May 30 12:47:39 selinux systemd[1]: Unit nginx.service entered failed state.
May 30 12:47:39 selinux systemd[1]: nginx.service failed.
```
##### &ensp;&ensp; Более подробный ход решения и вывод команд представлен в прилагающемся логе утилиты Script - z_1.log. #####
