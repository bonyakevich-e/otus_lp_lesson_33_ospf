### OTUS Linux Professional Lesson #33 Subject: Статическая и динамическая маршрутизация, OSPF

#### Цель домашнего задания:
Создать домашнюю сетевую лабораторию. Научится настраивать протокол OSPF в Linux-based системах.

#### Описание домашнего задания:
1. Развернуть 3 виртуальные машины
2. Объединить их разными vlan
- настроить OSPF между машинами на базе Quagga;
- изобразить ассиметричный роутинг;
- сделать один из линков "дорогим", но что бы при этом роутинг был симметричным.

__1. Разворачиваем 3 виртуальные машины__

Так как мы планируем настроить OSPF, все 3 виртуальные машины должны быть соединены между собой (разными VLAN), а также иметь одну (или несколько) доолнительных сетей, к которым, далее OSPF сформирует маршруты.

![image](https://github.com/user-attachments/assets/57067149-7951-412e-9323-98d15b248e8b)

```
$ vagrant up
```
Результатом выполнения данной команды будут 3 созданные виртуальные машины, которые соединены между собой сетями (10.0.10.0/30, 10.0.11.0/30 и 10.0.12.0/30). У каждого роутера есть дополнительная сеть:
- на router1 — 192.168.10.0/24
- на router2 — 192.168.20.0/24
- на router3 — 192.168.30.0/24
  
На данном этапе ping до дополнительных сетей (192.168.10-30.0/24) с соседних роутеров будет недоступен. 

__2.1. Настройка OSPF между машинами на базе пакета FRR__

1) Отключаем файерволл ufw и удаляем его из автозагрузки:
```
root@router1:~# systemctl stop ufw
root@router1:~# systemctl disable ufw
```
2) Устанавливаем пакет FRR:
```
root@router1:~# curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add -
root@router1:~# echo deb https://deb.frrouting.org/frr $(lsb_release -s -c) frr-stable > /etc/apt/sources.list.d/frr.list
root@router1:~# sudo apt update
root@router1:~# apt install frr frr-pythontools
```
3) Разрешаем (включаем) маршрутизацию транзитных пакетов:
```
root@router1:~# echo "net.ipv4.conf.all.forwarding = 1" >> /etc/sysctl.conf
root@router1:~# sysctl -p
```
4) Включаем демон ospfd в FRR. Для в файле _/etc/frr/daemons_ меняем параметры для пакетов ospfd на yes:
```
bgpd=no
ospfd=yes
ospf6d=yes
ripd=no
...
...
```
5) Для настройки OSPF нам потребуется создать файл /etc/frr/frr.conf который будет содержать в себе информацию о требуемых интерфейсах и OSPF. Для начала нам необходимо узнать имена интерфейсов и их адреса. Сделать это можно двумя способами:
- посмотреть с помощью команды `ip a | grep inet`:
```
root@router1:~# ip a | grep inet
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
    inet6 fe80::7b:48ff:fead:a30d/64 scope link 
    inet 10.0.10.1/30 brd 10.0.10.3 scope global enp0s8
    inet6 fe80::a00:27ff:febb:7c3c/64 scope link 
    inet 10.0.12.1/30 brd 10.0.12.3 scope global enp0s9
    inet6 fe80::a00:27ff:fe1f:3681/64 scope link 
    inet 192.168.10.1/24 brd 192.168.10.255 scope global enp0s10
    inet6 fe80::a00:27ff:fe24:e5ba/64 scope link 
```
- зайти в интерфейс FRR и посмотреть информацию об интерфейсах:
```
root@router1:~# vtysh

Hello, this is FRRouting (version 10.0.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# show interface brief
Interface       Status  VRF             Addresses
---------       ------  ---             ---------
enp0s3          up      default         10.0.2.15/24
                                        fe80::7b:48ff:fead:a30d/64
enp0s8          up      default         10.0.10.1/30
                                        fe80::a00:27ff:febb:7c3c/64
enp0s9          up      default         10.0.12.1/30
                                        fe80::a00:27ff:fe1f:3681/64
enp0s10         up      default         192.168.10.1/24
                                        fe80::a00:27ff:fe24:e5ba/64
lo              up      default         

```
В обоих примерах мы увидем имена сетевых интерфейсов, их ip-адреса и маски подсети. Исходя из схемы мы понимаем, что для настройки OSPF нам достаточно описать интерфейсы enp0s8, enp0s9, enp0s10.

Создаём файл /etc/frr/frr.conf и вносим в него следующую информацию:
```bash
# Указываем имя машины
hostname router1
log syslog informational
no ipv6 forwarding
service integrated-vtysh-config

# Добавляем информацию об интерфейсе enp0s8
interface enp0s8
 # Указываем имя интерфейса
 description r1-r2
 # Указываем ip-aдрес и маску
 ip address 10.0.10.1/30
 # Указываем параметр игнорирования MTU
 ip ospf mtu-ignore
 # Если потребуется, можно указать «стоимость» интерфейса
 # ip ospf cost 1000
 # Указываем параметры hello-интервала для OSPF пакетов
 ip ospf hello-interval 10
 # Указываем параметры dead-интервала для OSPF пакетов
 # Должно быть кратно предыдущему значению
 ip ospf dead-interval 30

interface enp0s9
 description r1-r3
 ip address 10.0.12.1/30
 ip ospf mtu-ignore
 !ip ospf cost 45
 ip ospf hello-interval 10
 ip ospf dead-interval 30

interface enp0s10
 description net_router1
 ip address 192.168.10.1/24
 ip ospf mtu-ignore
 !ip ospf cost 45
 ip ospf hello-interval 10
 ip ospf dead-interval 30

# Начало настройки OSPF
router ospf
 # Указываем router-id 
 router-id 1.1.1.1
 # Указываем сети, которые хотим анонсировать соседним роутерам
 network 10.0.10.0/30 area 0
 network 10.0.12.0/30 area 0
 network 192.168.10.0/24 area 0
 # Указываем адреса соседних роутеров
 neighbor 10.0.10.2
 neighbor 10.0.12.2

```
Вместо файла frr.conf мы можем задать данные параметры вручную из vtysh. Vtysh использует cisco-like команды. На хостах router2 и router3 также требуется настроить конфигруационные файлы, предварительно поменяв ip -адреса интерфейсов.
В ходе создания файла мы видим несколько OSPF-параметров, которые требуются для настройки:
- __hello-interval__ — интервал который указывает через сколько секунд протокол OSPF будет повторно отправлять запросы на другие роутеры. Данный интервал должен быть одинаковый на всех портах и роутерах, между которыми настроен OSPF. 
- __dead-interval__ — если в течении заданного времени роутер не отвечает на запросы, то он считается вышедшим из строя и пакеты уходят на другой роутер (если это возможно). Значение должно быть кратно hello-интервалу. Данный интервал должен быть одинаковый на всех портах и роутерах, между которыми настроен OSPF.
- __router-id__ — идентификатор маршрутизатора (необязательный параметр), если данный параметр задан, то роутеры определяют свои роли по данному параметру. Если данный идентификатор не задан, то роли маршрутизаторов определяются с помощью Loopback-интерфейса или самого большого ip-адреса на роутере.

6) После создания файлов /etc/frr/frr.conf и /etc/frr/daemons нужно проверить, что владельцем файла является пользователь frr. Группа файла также должна быть frr. Должны быть установленны следующие права:
- у владельца на чтение и запись
- у группы только на чтение
Если права или владелец файла указан неправильно, то нужно поменять владельца и назначить правильные права.

7) Перезапускаем FRR и добавляем его в автозагрузку:
```
root@router1:~# systemctl enable --now frr
```
Если мы правильно настроили OSPF, то с любого хоста нам должны быть доступны сети:
- 192.168.10.0/24
- 192.168.20.0/24
- 192.168.30.0/24
- 10.0.10.0/30 
- 10.0.11.0/30
- 10.0.13.0/30

Убедиться в этом можно пропинговав интерфейсы роутеров и/или посмотреть таблицу маршрутизации на наличие указанных маршрутов:
```
root@router1:~# ip r
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100 
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 
10.0.2.2 dev enp0s3 proto dhcp scope link src 10.0.2.15 metric 100 
10.0.10.0/30 dev enp0s8 proto kernel scope link src 10.0.10.1 
10.0.11.0/30 nhid 32 proto ospf metric 20 
	nexthop via 10.0.10.2 dev enp0s8 weight 1 
	nexthop via 10.0.12.2 dev enp0s9 weight 1 
10.0.12.0/30 dev enp0s9 proto kernel scope link src 10.0.12.1 
192.168.10.0/24 dev enp0s10 proto kernel scope link src 192.168.10.1 
192.168.20.0/24 nhid 28 via 10.0.10.2 dev enp0s8 proto ospf metric 20 
192.168.30.0/24 nhid 33 via 10.0.12.2 dev enp0s9 proto ospf metric 20 
```
```
root@router1:~# ping 10.0.11.1
PING 10.0.11.1 (10.0.11.1) 56(84) bytes of data.
64 bytes from 10.0.11.1: icmp_seq=1 ttl=64 time=1.12 ms
64 bytes from 10.0.11.1: icmp_seq=2 ttl=64 time=1.09 ms
64 bytes from 10.0.11.1: icmp_seq=3 ttl=64 time=1.08 ms
```
```
root@router1:~# vtysh

Hello, this is FRRouting (version 10.0.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# show ip route ospf
Codes: K - kernel route, C - connected, L - local, S - static,
       R - RIP, O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric, t - Table-Direct,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/100] is directly connected, enp0s8, weight 1, 13:07:43
O>* 10.0.11.0/30 [110/200] via 10.0.10.2, enp0s8, weight 1, 13:06:43
  *                        via 10.0.12.2, enp0s9, weight 1, 13:06:43
O   10.0.12.0/30 [110/100] is directly connected, enp0s9, weight 1, 13:07:43
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1, 13:07:43
O>* 192.168.20.0/24 [110/200] via 10.0.10.2, enp0s8, weight 1, 13:06:58
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, weight 1, 13:06:43
```

__2.2. Настройка ассиметричного роутинга__

Разрешаем ассиметричную маршрутизацию:
```
root@router1:~# sysctl net.ipv4.conf.all.rp_filter=0
```
Выбираем один из роутеров, на котором изменим «стоимость интерфейса». Например, поменяем стоимость интерфейса enp0s8 на router1:
```
root@router1:~# vtysh

Hello, this is FRRouting (version 10.0.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# conf t
router1(config)# int enp0s8
router1(config-if)# ip ospf cost 1000
router1(config-if)# exit
router1(config)# exit
router1# show ip route ospf
Codes: K - kernel route, C - connected, L - local, S - static,
       R - RIP, O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric, t - Table-Direct,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:00:40
O>* 10.0.11.0/30 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:00:40
O   10.0.12.0/30 [110/100] is directly connected, enp0s9, weight 1, 13:19:55
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1, 13:19:55
O>* 192.168.20.0/24 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:00:40
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, weight 1, 13:18:55
```
Видим пакеты с router1 к сетям router2 теперь пойдут через router3.  
Ответные пакеты c router2 к router1 пойдут по прежнему пути, сразу на router1:
```
root@router2:~# vtysh

Hello, this is FRRouting (version 10.0.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router2# show ip route ospf 
Codes: K - kernel route, C - connected, L - local, S - static,
       R - RIP, O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric, t - Table-Direct,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/100] is directly connected, enp0s8, weight 1, 13:31:45
O   10.0.11.0/30 [110/100] is directly connected, enp0s9, weight 1, 13:31:55
O>* 10.0.12.0/30 [110/200] via 10.0.10.1, enp0s8, weight 1, 13:31:01
  *                        via 10.0.11.1, enp0s9, weight 1, 13:31:01
O>* 192.168.10.0/24 [110/200] via 10.0.10.1, enp0s8, weight 1, 13:31:35
O   192.168.20.0/24 [110/100] is directly connected, enp0s10, weight 1, 13:31:55
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, enp0s9, weight 1, 13:31:01
```
Для проверки запускаем на router1 пинг от 192.168.10.1 до 192.168.20.1:
```
root@router1:~# ping -I 192.168.10.1 192.168.20.1
```
На router2 запускаем tcpdump, который будет смотреть трафик только на порту enp0s9:
```
root@router2:~# tcpdump -i enp0s9
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
07:32:24.272922 IP 192.168.10.1 > router2: ICMP echo request, id 4, seq 32, length 64
07:32:25.275290 IP 192.168.10.1 > router2: ICMP echo request, id 4, seq 33, length 64
07:32:25.687058 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
07:32:26.277340 IP 192.168.10.1 > router2: ICMP echo request, id 4, seq 34, length 64
07:32:27.220286 IP 10.0.11.1 > ospf-all.mcast.net: OSPFv2, Hello, length 48
07:32:27.279791 IP 192.168.10.1 > router2: ICMP echo request, id 4, seq 35, length 64
07:32:28.284012 IP 192.168.10.1 > router2: ICMP echo request, id 4, seq 36, length 64
07:32:29.287697 IP 192.168.10.1 > router2: ICMP echo request, id 4, seq 37, length 64
07:32:30.289778 IP 192.168.10.1 > router2: ICMP echo request, id 4, seq 38, length 64
07:32:31.294003 IP 192.168.10.1 > router2: ICMP echo request, id 4, seq 39, length 64
```
Видим что запросы приходят на интерфейс enp0s9.

На router2 запускаем tcpdump, который будет смотреть трафик только на порту enp0s8:
```
root@router2:~# tcpdump -i enp0s8
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
07:34:21.611560 IP router2 > 192.168.10.1: ICMP echo reply, id 5, seq 6, length 64
07:34:21.780555 ARP, Request who-has 10.0.10.1 tell router2, length 28
07:34:21.781597 ARP, Reply 10.0.10.1 is-at 08:00:27:bb:7c:3c (oui Unknown), length 46
07:34:22.614771 IP router2 > 192.168.10.1: ICMP echo reply, id 5, seq 7, length 64
07:34:23.639192 IP router2 > 192.168.10.1: ICMP echo reply, id 5, seq 8, length 64
07:34:24.664133 IP router2 > 192.168.10.1: ICMP echo reply, id 5, seq 9, length 64
07:34:25.666114 IP router2 > 192.168.10.1: ICMP echo reply, id 5, seq 10, length 64
07:34:25.820337 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
07:34:26.679309 IP router2 > 192.168.10.1: ICMP echo reply, id 5, seq 11, length 64
```
Видми что ответы уходят с интерфейса enp0s9. 

Таким образом мы имеем ассиметричную маршрутизацию. 

__2.3. Настройка симметичного роутинга__ 

Так как у нас уже есть один «дорогой» интерфейс, нам потребуется добавить ещё один дорогой интерфейс, чтобы у нас перестала работать ассиметричная маршрутизация. 

Так как в прошлом задании мы заметили что router2 будет отправлять обратно трафик через порт enp0s8, мы также должны сделать его дорогим и далее проверить, что теперь используется симметричная маршрутизация:

Поменяем стоимость интерфейса enp0s8 на router2:
```
router2# conf t
router2(config)# interface enp0s8
router2(config-if)# ip ospf cost 1000
router2(config-if)# end
router2# show ip route ospf 
Codes: K - kernel route, C - connected, L - local, S - static,
       R - RIP, O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric, t - Table-Direct,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/1000] is directly connected, enp0s8, weight 1, 00:00:12
O   10.0.11.0/30 [110/100] is directly connected, enp0s9, weight 1, 13:47:05
O>* 10.0.12.0/30 [110/200] via 10.0.11.1, enp0s9, weight 1, 00:00:12
O>* 192.168.10.0/24 [110/300] via 10.0.11.1, enp0s9, weight 1, 00:00:12
O   192.168.20.0/24 [110/100] is directly connected, enp0s10, weight 1, 13:47:05
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, enp0s9, weight 1, 13:46:11
```
Видим, что маршрут до сети 192.168.10.0/30 пойдёт через router2.
