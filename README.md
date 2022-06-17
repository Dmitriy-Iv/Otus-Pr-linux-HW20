# **Введение**

В данном домашнем задании нам необходимо получить практический начальный опыт работы с Iptables.

---

# Настройка тест лабы
1. Для выполнения данного ДЗ был взят стенд из 17 ДЗ. Убраны лишние сервера, и добавлен сервер inetRouter2. В результате получилась вот такая адресация у серверов
![alt text](/screenshots/hw20-1.PNG?raw=true "Screenshot1") 
2. Для запуска стенда выполняем следующее.
```
git clone https://github.com/Dmitriy-Iv/Otus-Pr-linux-HW20.git
cd Otus-Pr-linux-HW20/Ansible/
vagrant up
```

# Реализация Port Knocking
1. За основу была взята статья - `https://otus.ru/nest/post/267/`. Соотвественно на inetRouter получился такой конфиг для Iptables. Данное правило добавлено `-A INPUT -p tcp -s 192.168.56.0/24 -m tcp --dport 22 -j ACCEPT` чтобы работал ansible с хостовой машины.
```
# Generated by iptables-save v1.4.21 on Wed May 18 20:21:02 2022
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING ! -d 192.168.0.0/16 -o eth0 -j MASQUERADE
COMMIT
# Completed on Wed May 18 20:21:02 2022
# Generated by iptables-save v1.4.21 on Wed May 18 20:21:02 2022
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:TRAFFIC - [0:0]
:SSH-INPUT - [0:0]
:SSH-INPUTTWO - [0:0]
-A INPUT -i lo -j ACCEPT
-A INPUT -p tcp -s 192.168.56.0/24 -m tcp --dport 22 -j ACCEPT
-A INPUT -j TRAFFIC
-A TRAFFIC -p icmp --icmp-type any -j ACCEPT
-A TRAFFIC -m state --state ESTABLISHED,RELATED -j ACCEPT
-A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 22 -m recent --rcheck --seconds 30 --name SSH2 -j ACCEPT
-A TRAFFIC -m state --state NEW -m tcp -p tcp -m recent --name SSH2 --remove -j DROP
-A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 9991 -m recent --rcheck --name SSH1 -j SSH-INPUTTWO
-A TRAFFIC -m state --state NEW -m tcp -p tcp -m recent --name SSH1 --remove -j DROP
-A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 7777 -m recent --rcheck --name SSH0 -j SSH-INPUT
-A TRAFFIC -m state --state NEW -m tcp -p tcp -m recent --name SSH0 --remove -j DROP
-A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 8881 -m recent --name SSH0 --set -j DROP
-A SSH-INPUT -m recent --name SSH1 --set -j DROP
-A SSH-INPUTTWO -m recent --name SSH2 --set -j DROP
-A TRAFFIC -j DROP
COMMIT
```
2. Проверяем, как это отрабатывает с centralRouter.
```
dima@Test-Ubuntu-1:~/otus/my-hw20/Ansible$ vagrant ssh centralRouter
Last login: Fri Jun 17 17:58:05 2022 from 10.0.2.2

[vagrant@centralRouter ~]$ sudo -i

[root@centralRouter ~]#  ssh vagrant@192.168.255.1
^C
[root@centralRouter ~]# ./knock.sh 192.168.255.1 8881 7777 9991

Starting Nmap 6.40 ( http://nmap.org ) at 2022-06-17 18:00 UTC
Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.255.1
Host is up (0.00046s latency).
PORT     STATE    SERVICE
8881/tcp filtered unknown
MAC Address: 08:00:27:2C:B1:A9 (Cadmus Computer Systems)

Nmap done: 1 IP address (1 host up) scanned in 0.44 seconds

Starting Nmap 6.40 ( http://nmap.org ) at 2022-06-17 18:00 UTC
Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.255.1
Host is up (0.00051s latency).
PORT     STATE    SERVICE
7777/tcp filtered cbt
MAC Address: 08:00:27:2C:B1:A9 (Cadmus Computer Systems)

Nmap done: 1 IP address (1 host up) scanned in 0.34 seconds

Starting Nmap 6.40 ( http://nmap.org ) at 2022-06-17 18:00 UTC
Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.255.1
Host is up (0.00042s latency).
PORT     STATE    SERVICE
9991/tcp filtered issa
MAC Address: 08:00:27:2C:B1:A9 (Cadmus Computer Systems)

Nmap done: 1 IP address (1 host up) scanned in 0.34 seconds

[root@centralRouter ~]#  ssh vagrant@192.168.255.1
Warning: Permanently added '192.168.255.1' (ECDSA) to the list of known hosts.
vagrant@192.168.255.1's password:
Last login: Fri Jun 17 17:35:30 2022 from 192.168.56.1
```
Из вывода выше видно, что просто подключится по ssh у нас не получилось, только после запуска скрипта, который стучит по портам 8881 7777 9991 у нас появляется возможность в течении 30 секунд открыть ssh подключение.

# Проброс порта 8080 с inetRouter2 на порт 80 centralServer
1. Топоплогия сети оставалась прежней, как и в ДЗ 17. Только теперь к centralRouter подключили ещё inetRouter2. Default route как и раньше с centralServer идёт на centralRouter и далее на inetRouter (это видно при запуске traceroute c centralServer - вывод есть в playbook). Для реализации проброса порта 8080 на inetRouter2 был использован интерфейс eth2 (ip 192.168.56.13 host-only - он же используется для ansible), который дальше отправляет запрос на 80 порт через интерфейс eth1 (ip 192.168.0.34) на сервер centralRouter (ip 192.168.0.33), который перенапрявляет запрос уже на centralServer (ip 192.168.0.2). Для этого в Iptables были внесены следующие изменения.
```
iptables -t nat -A PREROUTING -p tcp -d 192.168.56.13 --dport 8080 -j DNAT --to-destination 192.168.0.2:80
iptables -t nat -A POSTROUTING -s 192.168.56.0/24 -d 192.168.0.2 -j MASQUERADE
iptables -A FORWARD -i eth2 -d 192.168.0.2 -p tcp --dport 80 -j ACCEP
```
2. Проверка работоспособности проброса c хостовой машины.
```
dima@Test-Ubuntu-1:~/otus/my-hw20/Ansible$ curl 192.168.56.13:8080
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
        line-height: 1.25em;
        margin: 0 4% 0 4%;
        }
......................................
```