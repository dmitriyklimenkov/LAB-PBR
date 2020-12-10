# LAB-PBR

# Лабораторная работа: Маршрутизация на основе политик (PBR).
# Задание:
- Настроите политику маршрутизации для сетей офиса
- Распределите трафик между двумя линками с провайдером
- Настроите отслеживание линка через технологию IP SLA
- Настройте для офиса Лабытнанги маршрут по-умолчанию

# Решение:

# Схема сегмента сети:

![](https://github.com/dmitriyklimenkov/LAB-PBR/blob/main/%D0%A1%D1%85%D0%B5%D0%BC%D0%B0%20PBR.PNG)

# 1. Конфигурация PBR IPv4 на R28.
Адресация на портах была настроена ранее. Настроим IP SLA:
```
ip sla auto discovery
ip sla 1
 icmp-echo 194.14.123.33 source-interface Ethernet0/0
 threshold 1000
 timeout 1500
 frequency 3
ip sla schedule 1 life forever start-time now

ip sla 2
 icmp-echo 194.14.123.29 source-interface Ethernet0/1
 threshold 1000
 timeout 1500
 frequency 3
ip sla schedule 2 life forever start-time now
```

Настроим track:
```
track 10 ip sla 1 reachability
 delay down 10 up 5
track 20 ip sla 2 reachability
 delay down 10 up 5
 ```

IP SLA 1 будет проверять доступность по IPv4 адресу R26.
IP SLA 2 будет проверять доступность по IPv4 адресу R25.
Track 10 отслеживает состояние IP SLA 1.
Track 20 отслеживает состояние IP SLA 2.
Пропишем два маршрута IPv4 по умолчанию.
```
ip route 0.0.0.0 0.0.0.0 194.14.123.33 track 10
ip route 0.0.0.0 0.0.0.0 194.14.123.29 track 20
```
Настроим ACL для двух локальных подсетей.
```
ip access-list extended acl_nat1
 permit ip 201.193.45.0 0.0.0.63 any
ip access-list extended acl_nat2
 permit ip 201.193.45.64 0.0.0.63 any
```

Настроим Route-map.
```
route-map TO_R25 permit 10
 match ip address acl_nat2
 match interface Ethernet0/2.4
 set ip next-hop verify-availability 194.14.123.29 1 track 20
!
route-map TO_R26 permit 10
 match ip address acl_nat1
 match interface Ethernet0/2.3
 set ip next-hop verify-availability 194.14.123.33 1 track 10
```
И пропишем на сабинтерфейсах локальной сети.
```
interface Ethernet0/2.3
 encapsulation dot1Q 3
 ip address 201.193.45.1 255.255.255.192
 ip policy route-map TO_R26
!
interface Ethernet0/2.4
 encapsulation dot1Q 4
 ip address 201.193.45.65 255.255.255.192
 ip policy route-map TO_R25

```
VPC в локальной сети находятся в разных VLAN. Данные VPC-30 будут проходить через R26, данные VPC-31 будут проходить через R25. При недоступности одного из линков, все данные будут идти через активный линк.

# Проверка PBR IPv4.
Для проверки с VPC 31 пингуем 8.8.8.8 и наблюдаем через wireshark за интерфейсами e0/1 и e0/0. Когда активный линк R28-R25, пакеты идут по этому маршруту. Если линк R28-R25 погасить, пакеты пойдут через линк R28-R26, как и требуется по заданию.



# 2. Конфигурация PBR IPv6 на R28.
Настроим IP SLA:
```
ip sla 100
 icmp-echo 200C:C0FE:1111:90::2
 threshold 1000
 timeout 1500
 frequency 3
ip sla schedule 100 life forever start-time now
```

Настроим track:
```
track 100 ip sla 100 reachability
 delay down 10 up 5
```
IP SLA 100 будет проверять доступность по IPv6 адресу R26.
Track 100 IP SLA 100

Настроим маршрут по умолчанию через R26:
```
ipv6 route ::/64 200C:C0FE:1111:90::2
```

Для реализации автоматичского переключения на резервный канал используем EEM вместе с IP SLA. Ниже представлена конфигурация EEM:
```
event manager applet ISP1_UP
 event track 100 state up
 action 001 cli command "enable"
 action 002 cli command "configure terminal"
 action 003 cli command "no ipv6 route ::0/0 200c:c0fe:1111:80::2"
 action 004 cli command "ipv6 route ::0/0 200c:c0fe:1111:90::2"
 action 005 syslog msg "ISP1 is UP"
 action 006 cli command "end"
 action 007 cli command "exit"
 action 008 cli command "exit"
event manager applet ISP1_DOWN
 event track 100 state down
 action 001 cli command "enable"
 action 002 cli command "configure terminal"
 action 003 cli command "ipv6 route ::0/0 200c:c0fe:1111:80::2"
 action 004 cli command "no ipv6 route ::0/0 200c:c0fe:1111:90::2"
 action 005 syslog msg "ISP1 is DOWN"
 action 006 cli command "end"
 action 007 cli command "exit"
 action 008 cli command "exit"
```
В зависимости от состояния track 100, будут выполнятся определенный выше действия в конфигурации маршрутизатора


# 3. Настройка IPv4, IPv6 маршрутов по умолчанию на R27.
```
ip route 0.0.0.0 0.0.0.0 194.14.123.25
!
ipv6 route ::/0 200C:C0FE:1111:70::2
```
