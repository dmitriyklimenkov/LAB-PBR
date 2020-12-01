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
```

Настроим track:
```
track 10 ip sla 1 reachability
 delay down 10 up 5
 ```

IP SLA 1 будет проверять доступность по IPv4 адресу R26.
Track 10 отслеживает состояние IP SLA 1.
Пропишем два маршрута IPv4 по умолчанию, основным будет до R26, резервным до R25. Резервный маршрут пропишем с большей AD.
```
ip route 0.0.0.0 0.0.0.0 194.14.123.33 track 10
ip route 0.0.0.0 0.0.0.0 194.14.123.29 20
```
Добавление в конце основного маршрута track 10 говорит о том, что этот маршрут будет использоваться только тогда, когда состояние track 10 в UP.

# 2. Проверка PBR IPv4.
Проверим таблицу маршрутизации, когда основной линк в работает:

![](https://github.com/dmitriyklimenkov/LAB-PBR/blob/main/ipv4%20ip%20route1.PNG)

Видно, что маршрут по умолчанию настроен через R26.
Теперь на R26 отключим интерфейс e0/1.
Появилось сообщение:
```
TRACKING-5-STATE: 10 ip sla 1 reachability Up->Down
```
Проверим таблицу маршрутизации еще раз:

![](https://github.com/dmitriyklimenkov/LAB-PBR/blob/main/ipv4%20ip%20route2.PNG)

Видно, что маршрут по умолчанию теперь через R25.

# 3. Конфигурация PBR IPv6 на R28.
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

# 4. Проверка PBR IPv6.

Проверим таблицу маршрутизации, когда основной линк в работает:

![](https://github.com/dmitriyklimenkov/LAB-PBR/blob/main/ipv6%20ip%20route1.PNG)

Видно, что маршрут по умолчанию настроен через R26.
Теперь на R26 отключим интерфейс e0/1.

Появились сообщения: 
```
TRACKING-5-STATE: 100 ip sla 100 reachability Up->Down
HA_EM-6-LOG: ISP1_DOWN: ISP1 is DOWN
```
Проверим таблицу маршрутизации еще раз:

![](https://github.com/dmitriyklimenkov/LAB-PBR/blob/main/ipv6%20ip%20route2.PNG)

Видно, что маршрут по умолчанию теперь через R25.

# 5. Настройка IPv4, IPv6 маршрутов по умолчанию на R27.
```
ip route 0.0.0.0 0.0.0.0 194.14.123.25
!
ipv6 route ::/0 200C:C0FE:1111:70::2
```
