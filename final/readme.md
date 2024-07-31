# Проектная работа

## Цель:

- Моделирование сети дата-центра на оборудовании Juniper.

## План

- Разработка топологии
- Проектирование адресного пространства
- Проектирование сети Underlay
- Проектирование сети Overlay
- Проектирование клиентских подключений
- Реализация

## Разработка топология
При проектировании топологии сети нужно понимать сколько трафика должно проходить по ней.
В современных DC меньшую часть трафика занимает направление "север-юг" т.е. North-South, 
а большая часть занимает трафик "восток-запад" или "East-West". 
Сеть должна отвечать следующим основным требованиям:
- масштабируемость
- отказоустойчивость
- простота эксплуатации
- отсутствие переподписки

Если первые 3 пункта более-менее ясны, то про последний стоит немного сказать более подробно.
Пропускная способность от свитчей куда включается клиентское оборудование должна быть одинаковой на всем пути прохождения трафика.
Это в идеальном случае и не важно какой путь имеется ввиду North-South или East-West.
Смысл переподписки в том, что сервисные свитчи (leaf) могут иметь большую пропускную способность в сторону клиентов,
чем пропускная способность в сторону коммутационного уровня (Spine).

Например, схема с 50ю даунлинками по 10Гб/с и 3мя аплинками по 100Гб/с на Leaf — это фабрика без переподписки 
и даже при одновременной передаче данных всеми подключенными хостами — всегда хватит аплинков.

Всем указанным требованиям отвечает сеть CLOS (от имени Чарльза Клоза)
Leaf свитчи это свитчи с максимальным функционалом на нем будут находится все сервисные включения
Spine свитчи это свитчи с максимальной пропускной способностью. На этом уровне свитчей полностью отсуствуют клиентские включения.
Leaf свитч включается во все Spine свитчи. 
Нет подключений вида Spine-Spine, Leaf-Leaf.

![Пример сети CLOS](target_topo-1.avif "Пример сети CLOS")

Для простоты конфигурации и придерживания принципа "сделать все максимально просто", 
В качестве underlay и overlay будет использован eBGP и iBGP соответственно, а технологией туннелирования будет использован VxLAN.

## Проектирование адресного пространства

Распределение адресов для Loopback интерфейсов

| Device Type | IP Range | 
|:-------------|:----------|
| Leaf | 10.255.254.0/24|
| Spine | 10.255.255.0/24|

Распределение адресов для p2p интерфейсов<br />
Leaf-N <-> Spine | 10.0.n.l/30.<br />
Где:<br />
N - номер leaf свитча, <br />
L - номер линка, считатся от номера сети. 0, 4, 8, и т.д.<br />

На Leaf младший адрес<br />
На Spine старший адрес<br />


## Проектирование сети Underlay
BGP протокол имеет мощный функционал по управлению анонсами и с легкостью справится даже с самыми большими DC сетями.
Основной момент в Underlay сети такой, что каждый leaf свитч имеет свой уникальный ASN, а все Spine свитчи находятся в одной ASN.
leaf-1 65511, 
leaf-2 65512,
leaf-3 65513,
Spine-1, Spine-2 65501

eBGP сессия поднимается на point-to-point линках и принимают и анонсируют только loopback адреса, т.е. `family unicast`.
Между Spine свитчами нужно обеспечить обмен префиксов для этого необходимо включить `as-loop-in 1` или `as-override`, чтобы leaf свитч принимая префикс от spine-2 и анонсируя их в сторону spine-1, spine-1 их принимал. т.е. отключаем встроенную в BGP механизм предотвращения петель, ровно на одно вхождение.
Далее чтобы свитч в принципе начала анонсировать маршруты из той же AS с кем настроено соседство нужно включить опцию `advertise-peer-as`

Поскольку у нас есть множество путей между leaf свитчами, трафик должен быть разбалансирован между всеми имеющимися маршрутами, а поскольку у нас eBGP и с разными внешними AS, то необходимо включить `multipath multiple-as`, что позволит использовать ECMP.

![Сеть Underlay](topology-bgp.png "Сеть Underlay")


## Проектирование сети Overlay
Overlay сеть так же будет работать на основе протокола BGP, поскольку underlay нам обеспечил связность loopback адресов, то мы можем с легкостью использовать iBGP.
iBGP протокол требует full-mesh связности между всеми узлами сети, либо использовать route-reflector который упростит задачу масштабируемости и эксплуатации.
iBGP AS 4210000001
Настройка overlay сети достаточно проста и она включает в себя только одну address family это `family evpn signaling`
которое позволяет нам динамически обновлять статус VTEP'ов.

При настройке iBGP нужно включить функцию `route-reflector` на spine свитчах и добавить их в один кластер.
iBGP сессия между всем Leaf и Spine, а spine свитчи только между собой.


## Проектирование клиентских подключений
Проектируемая сеть позволяет обеспечивать сервисы L2, L3 для клиентских включений.
L3 на основе традиционного L3 VPN с возможность выхода в Интернет через EVPN Type-5
L2 на основе EVPN
Резервирование обеспечивается на основе ESI LAG т.е. когда клиент одновременно включается в два и более leaf свитчей и 
организует LAG со свой стороны.


## Реализация

#### Пример конфигурации Underlay

leaf-1
```
set policy-options policy-statement BGP_LOOPBACK0 term TERM1 from protocol direct
set policy-options policy-statement BGP_LOOPBACK0 term TERM1 from route-filter 0.0.0.0/0 prefix-length-range /32-/32
set policy-options policy-statement BGP_LOOPBACK0 term TERM1 then accept
set protocols bgp group UNDERLAY export BGP_LOOPBACK0

set policy-options policy-statement PFE-ECMP then load-balance per-packet
set routing-options forwarding-table export PFE-ECMP

set protocols bgp group UNDERLAY type external
set protocols bgp group UNDERLAY advertise-peer-as
set protocols bgp group UNDERLAY family inet unicast
set protocols bgp group UNDERLAY export BGP_LOOPBACK0
set protocols bgp group UNDERLAY peer-as 65501
set protocols bgp group UNDERLAY local-as 65511
set protocols bgp group UNDERLAY multipath multiple-as
set protocols bgp group UNDERLAY as-override
set protocols bgp group UNDERLAY neighbor 10.0.1.2
set protocols bgp group UNDERLAY neighbor 10.0.1.6

```
spine-1
```
set protocols bgp group UNDERLAY type external
set protocols bgp group UNDERLAY local-as 65501
set protocols bgp group UNDERLAY neighbor 10.0.1.1 peer-as 65511
set protocols bgp group UNDERLAY neighbor 10.0.2.1 peer-as 65512
set protocols bgp group UNDERLAY neighbor 10.0.3.1 peer-as 65513
set protocols bgp group UNDERLAY hold-time 10
set protocols bgp group UNDERLAY family inet unicast
set protocols bgp group UNDERLAY multipath multiple-as

set policy-options policy-statement PFE-ECMP then load-balance per-packet
set routing-options forwarding-table export PFE-ECMP
```

#### Пример конфигурации Overlay
Leaf-1
```
set routing-options autonomous-system 4210000001
set protocols bgp group OVERLAY type internal
set protocols bgp group OVERLAY local-address 10.255.254.1
set protocols bgp group OVERLAY family evpn signaling
set protocols bgp group OVERLAY neighbor 10.255.255.1
set protocols bgp group OVERLAY neighbor 10.255.255.2
```
Spine-1
```
set routing-options autonomous-system 4210000001
set protocols bgp group OVERLAY type internal
set protocols bgp group OVERLAY local-address 10.255.255.1
set protocols bgp group OVERLAY family evpn signaling
set protocols bgp group OVERLAY cluster 10.255.250.1
set protocols bgp group OVERLAY multipath
set protocols bgp group OVERLAY neighbor 10.255.254.1
set protocols bgp group OVERLAY neighbor 10.255.254.2
set protocols bgp group OVERLAY neighbor 10.255.254.3
```


#### Пример конфигурации L2 EVPN
Leaf-1
```
set interfaces ge-0/0/2 vlan-tagging
set interfaces ge-0/0/2 encapsulation flexible-ethernet-services
set interfaces ge-0/0/2 unit 0 encapsulation vlan-bridge
set interfaces ge-0/0/2 unit 0 vlan-id 10

set routing-instances EVPN_10 vtep-source-interface lo0.0
set routing-instances EVPN_10 instance-type evpn
set routing-instances EVPN_10 vlan-id 10
set routing-instances EVPN_10 interface ge-0/0/2.0
set routing-instances EVPN_10 vxlan vni 10010
set routing-instances EVPN_10 route-distinguisher 10.255.254.1:1
set routing-instances EVPN_10 vrf-target target:1234:1
set routing-instances EVPN_10 protocols evpn encapsulation vxlan
```

Leaf-3
```
set interfaces ge-0/0/2 vlan-tagging
set interfaces ge-0/0/2 encapsulation flexible-ethernet-services
set interfaces ge-0/0/2 unit 0 encapsulation vlan-bridge
set interfaces ge-0/0/2 unit 0 vlan-id 10

set routing-instances EVPN_10 vtep-source-interface lo0.0
set routing-instances EVPN_10 instance-type evpn
set routing-instances EVPN_10 vlan-id 10
set routing-instances EVPN_10 interface ge-0/0/2.0
set routing-instances EVPN_10 vxlan vni 10010
set routing-instances EVPN_10 route-distinguisher 10.255.254.3:1
set routing-instances EVPN_10 vrf-target target:1234:1
set routing-instances EVPN_10 protocols evpn encapsulation vxlan
```

#### Пример конфигурации L3 EVPN

Leaf-1
```
set interfaces ge-0/0/2 flexible-vlan-tagging
set interfaces ge-0/0/2 encapsulation extended-vlan-bridge
set interfaces ge-0/0/2 unit 10 vlan-id 10
set interfaces irb unit 10 virtual-gateway-accept-data
set interfaces irb unit 10 family inet address 10.5.5.100/24 virtual-gateway-address 10.5.5.254

set policy-options policy-statement mac10-import term local-orig from community v10
set policy-options policy-statement mac10-import term local-orig then accept
set policy-options policy-statement mac10-import term v20-orig from community v20
set policy-options policy-statement mac10-import term v20-orig then accept
set policy-options policy-statement mac10-import term v30-orig from community v30
set policy-options policy-statement mac10-import term v30-orig then accept

set policy-options policy-statement vrf10-import from community v10-vrf
set policy-options policy-statement vrf10-import from community v20-vrf
set policy-options policy-statement vrf10-import from community v30-vrf
set policy-options policy-statement vrf10-import then accept

set routing-instances v10 instance-type mac-vrf
set routing-instances v10 protocols evpn encapsulation vxlan
set routing-instances v10 protocols evpn extended-vni-list 10010
set routing-instances v10 vtep-source-interface lo0.0
set routing-instances v10 bridge-domains v10 vlan-id 10
set routing-instances v10 bridge-domains v10 interface ge-0/0/2.10
set routing-instances v10 bridge-domains v10 routing-interface irb.10
set routing-instances v10 bridge-domains v10 vxlan vni 10010
set routing-instances v10 service-type vlan-based
set routing-instances v10 route-distinguisher 10.255.254.1:1
set routing-instances v10 vrf-import mac10-import
set routing-instances v10 vrf-target target:42011:10
set routing-instances v10-vrf instance-type vrf
set routing-instances v10-vrf routing-options auto-export
set routing-instances v10-vrf protocols evpn irb-symmetric-routing vni 9910
set routing-instances v10-vrf interface irb.10
set routing-instances v10-vrf route-distinguisher 10.255.254.1:100
set routing-instances v10-vrf vrf-import vrf10-import
set routing-instances v10-vrf vrf-target target:42011:1010
set routing-instances v10-vrf vrf-table-label
```

Leaf-3
```
set interfaces ge-0/0/2 flexible-vlan-tagging
set interfaces ge-0/0/2 encapsulation extended-vlan-bridge
set interfaces ge-0/0/2 unit 10 vlan-id 10
set interfaces irb unit 10 virtual-gateway-accept-data
set interfaces irb unit 10 family inet address 10.5.5.200/24 virtual-gateway-address 10.5.5.254

set policy-options policy-statement mac10-import term local-orig from community v10
set policy-options policy-statement mac10-import term local-orig then accept
set policy-options policy-statement mac10-import term v20-orig from community v20
set policy-options policy-statement mac10-import term v20-orig then accept
set policy-options policy-statement mac10-import term v30-orig from community v30
set policy-options policy-statement mac10-import term v30-orig then accept

set policy-options policy-statement vrf10-import from community v10-vrf
set policy-options policy-statement vrf10-import from community v20-vrf
set policy-options policy-statement vrf10-import from community v30-vrf
set policy-options policy-statement vrf10-import then accept

set routing-instances v10 instance-type mac-vrf
set routing-instances v10 protocols evpn encapsulation vxlan
set routing-instances v10 protocols evpn extended-vni-list 10010
set routing-instances v10 vtep-source-interface lo0.0
set routing-instances v10 bridge-domains v10 vlan-id 10
set routing-instances v10 bridge-domains v10 interface ge-0/0/2.10
set routing-instances v10 bridge-domains v10 routing-interface irb.10
set routing-instances v10 bridge-domains v10 vxlan vni 10010
set routing-instances v10 service-type vlan-based
set routing-instances v10 route-distinguisher 10.255.254.3:1
set routing-instances v10 vrf-import mac10-import
set routing-instances v10 vrf-target target:42011:10
set routing-instances v10-vrf instance-type vrf
set routing-instances v10-vrf routing-options auto-export
set routing-instances v10-vrf protocols evpn irb-symmetric-routing vni 9910
set routing-instances v10-vrf interface irb.10
set routing-instances v10-vrf route-distinguisher 10.255.254.3:100
set routing-instances v10-vrf vrf-import vrf10-import
set routing-instances v10-vrf vrf-target target:42011:1010
set routing-instances v10-vrf vrf-table-label
```

Полная конфигурация в папке device-configs.

### Финальная конфигурация сети 

![Целевая сеть](topology-final.png "Целевая сеть")

Как видно из модели есть различные клиенты в различных vlan.
Один клиент подключен с помощью ESI LAG сразу в два свитча Leaf-1, Leaf-2.
Leaf-3 имеет внешние подключение к "Интернету", посредством передачи EVPN Type-5 маршрутов.

### BGP соседства Underlay

Пример вывода BGP соседств с leaf-3
```
root@leaf-3> show bgp summary 
Groups: 1 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0
                       4          4          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.3.2              65501        181        177       0       0        8:55 2/2/2/0              0/0/0/0
10.0.3.6              65501         19         17       0       0          44 2/2/2/0              0/0/0/0
```

Пример вывода BGP соседств со spine-1
```
root@spine-1> show bgp summary 
Groups: 1 Peers: 3 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0
                       3          3          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.1.1              65511        294        262       0       0       13:07 1/1/1/0              0/0/0/0
10.0.2.1              65512        163        165       0       0        8:13 1/1/1/0              0/0/0/0
10.0.3.1              65513        181        183       0       0        9:08 1/1/1/0              0/0/0/0
```


### BGP соседства Overlay

```
root@leaf-3> show bgp summary
Threading mode: BGP I/O
Groups: 2 Peers: 4 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0
                       8          6          0          0          0          0
bgp.evpn.0
                      12          6          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.3.2              65501        992        998       0       0       45:02 Establ
  inet.0: 3/4/4/0
10.0.3.6              65501        994        997       0       0       45:12 Establ
  inet.0: 3/4/4/0
10.255.255.1     4210000001       1337       1339       0       0     1:00:29 Establ
  EVPN_10.evpn.0: 3/3/3/0
  EVPN_20.evpn.0: 3/3/3/0
  __default_evpn__.evpn.0: 0/0/0/0
  bgp.evpn.0: 6/6/6/0
10.255.255.2     4210000001       1339       1342       0       0     1:00:39 Establ
  EVPN_10.evpn.0: 0/3/3/0
  EVPN_20.evpn.0: 0/3/3/0
  __default_evpn__.evpn.0: 0/0/0/0
  bgp.evpn.0: 0/6/6/0
```


```
root@spine-1> show bgp summary
Threading mode: BGP I/O
Groups: 3 Peers: 7 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0
                      10          6          0          0          0          0
bgp.evpn.0
                      12         12          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.1.1              65511       1044       1035       0       2       46:57 Establ
  inet.0: 2/3/3/0
10.0.2.1              65512       1014       1012       0       1       45:51 Establ
  inet.0: 2/3/3/0
10.0.3.1              65513       1008       1001       0       1       45:28 Establ
  inet.0: 2/4/4/0
10.255.254.1     4210000001       1031       1025       0       2       46:30 Establ
  bgp.evpn.0: 3/3/3/0
10.255.254.2     4210000001       1354       1363       0       0     1:01:15 Establ
  bgp.evpn.0: 3/3/3/0
10.255.254.3     4210000001       1349       1345       0       0     1:00:55 Establ
  bgp.evpn.0: 6/6/6/0
10.255.255.2     4210000001        119        118       0       0       46:55 Establ
  bgp.evpn.0: 0/0/0/0
```

Анонсирование и получение MAC/IP с Spine-1 в сторону Leaf-1
```
root@spine-1> show route advertising-protocol bgp 10.255.254.1

bgp.evpn.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  2:10.255.254.2:2::0::aa:bb:cc:00:04:00/304 MAC/IP
*                         10.255.254.2                 100        I
  2:10.255.254.3:1::0::aa:bb:cc:00:0c:00/304 MAC/IP
*                         10.255.254.3                 100        I
  2:10.255.254.3:2::0::aa:bb:cc:00:02:00/304 MAC/IP
*                         10.255.254.3                 100        I
  2:10.255.254.2:2::0::aa:bb:cc:00:04:00::172.17.0.1/304 MAC/IP
*                         10.255.254.2                 100        I
  2:10.255.254.3:1::0::aa:bb:cc:00:0c:00::10.5.5.20/304 MAC/IP
*                         10.255.254.3                 100        I
  2:10.255.254.3:2::0::aa:bb:cc:00:02:00::172.17.0.2/304 MAC/IP
*                         10.255.254.3                 100        I
  3:10.255.254.2:2::0::10.255.254.2/248 IM
*                         10.255.254.2                 100        I
  3:10.255.254.3:1::0::10.255.254.3/248 IM
*                         10.255.254.3                 100        I
  3:10.255.254.3:2::0::10.255.254.3/248 IM
*                         10.255.254.3                 100        I

root@spine-1> show route receive-protocol bgp 10.255.254.1

inet.0: 11 destinations, 17 routes (11 active, 0 holddown, 0 hidden)

inet6.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)

bgp.evpn.0: 12 destinations, 12 routes (12 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  2:10.255.254.1:1::0::aa:bb:cc:00:0b:00/304 MAC/IP
*                         10.255.254.1                 100        I
  2:10.255.254.1:1::0::aa:bb:cc:00:0b:00::10.5.5.10/304 MAC/IP
*                         10.255.254.1                 100        I
  3:10.255.254.1:1::0::10.255.254.1/248 IM
*                         10.255.254.1                 100        I

root@spine-1>
```

### Связность всех между собой

Доступность R1->R3
```
R1#show ip int br
Interface                  IP-Address      OK? Method Status                Protocol
Ethernet0/0                unassigned      YES unset  up                    up
Ethernet0/0.10             10.5.5.10       YES manual up                    up
Ethernet0/1                unassigned      YES unset  administratively down down
Ethernet0/2                unassigned      YES unset  administratively down down
Ethernet0/3                unassigned      YES unset  administratively down down
R1#ping 10.5.5.20
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.5.5.20, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 4/5/8 ms
R1#

```
Доступность R2->R4
```
R2#show ip int br
Interface                  IP-Address      OK? Method Status                Protocol
Ethernet0/0                unassigned      YES unset  up                    up
Ethernet0/0.20             172.17.0.1      YES manual up                    up
Ethernet0/1                unassigned      YES unset  administratively down down
Ethernet0/2                unassigned      YES unset  administratively down down
Ethernet0/3                unassigned      YES unset  administratively down down
R2#ping 172.17.0.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.17.0.2, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 4/5/10 ms
```

```
root@leaf-3> show evpn instance EVPN_10 extensive
Instance: EVPN_10
  Route Distinguisher: 10.255.254.3:1
  VLAN ID: 10
  Encapsulation type: VXLAN
  Duplicate MAC detection threshold: 5
  Duplicate MAC detection window: 180
  MAC database status                     Local  Remote
    MAC advertisements:                       1       1
    MAC+IP advertisements:                    1       1
    Default gateway MAC advertisements:       0       0
  Number of local interfaces: 2 (2 up)
    Interface name  ESI                            Mode             Status     AC-Role
    .local..8       00:00:00:00:00:00:00:00:00:00  single-homed     Up         Root
    ge-0/0/2.0      00:00:00:00:00:00:00:00:00:00  single-homed     Up         Root
  Number of IRB interfaces: 0 (0 up)
  Number of protect interfaces: 0
  Number of bridge domains: 1
    VLAN  Domain ID   Intfs / up    IRB intf   Mode      MAC sync  IM route label  IPv4 SG sync  IPv4 IM core nexthop  IPv6 SG sync  IPv6 IM core nexthop
    10    10010          1    1                Extended         Enabled   10010           Disabled                    Disabled
  Number of neighbors: 1
    Address               MAC    MAC+IP        AD        IM        ES Leaf-label
    10.255.254.1            1         1         0         1         0
  Number of ethernet segments: 0
  Router-ID: 10.255.254.3
  Source VTEP interface IP: 10.255.254.3
  SMET Forwarding: Disabled
```


Детальное описание маршрута. т.е.VxLAN туннель от leaf-3 (10.255.254.3) будет построен до leaf-1(10.255.254.1)
```
root@leaf-3> show route evpn-mac-address aa:bb:cc:00:0b:00 table EVPN_10 detail

EVPN_10.evpn.0: 6 destinations, 9 routes (6 active, 0 holddown, 0 hidden)
2:10.255.254.1:1::0::aa:bb:cc:00:0b:00/304 MAC/IP (2 entries, 1 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 10.255.254.1:1
                Next hop type: Indirect, Next hop index: 0
                Address: 0xce32a70
                Next-hop reference count: 12
                Source: 10.255.255.1
                Protocol next hop: 10.255.254.1
                Indirect next hop: 0x2 no-forward INH Session ID: 0x0
                State: <Secondary Active Int Ext>
                Local AS: 4210000001 Peer AS: 4210000001
                Age: 7:28       Metric2: 0
                Validation State: unverified
                Task: BGP_4210000001.10.255.255.1
                Announcement bits (1): 0-EVPN_10-evpn
                AS path: I  (Originator)
                Cluster list:  10.255.250.1
                Originator ID: 10.255.254.1
                Communities: target:1234:1 encapsulation:vxlan(0x8)
                Import Accepted
                Route Label: 10010
                ESI: 00:00:00:00:00:00:00:00:00:00
                Localpref: 100
                Router ID: 10.255.255.1
                Primary Routing Table bgp.evpn.0
         BGP    Preference: 170/-101
                Route Distinguisher: 10.255.254.1:1
                Next hop type: Indirect, Next hop index: 0
                Address: 0xce32a70
                Next-hop reference count: 12
                Source: 10.255.255.2
                Protocol next hop: 10.255.254.1
                Indirect next hop: 0x2 no-forward INH Session ID: 0x0
                State: <Secondary NotBest Int Ext Changed>
                Inactive reason: Not Best in its group - Update source
                Local AS: 4210000001 Peer AS: 4210000001
                Age: 7:28       Metric2: 0
                Validation State: unverified
                Task: BGP_4210000001.10.255.255.2
                AS path: I  (Originator)
                Cluster list:  10.255.250.1
                Originator ID: 10.255.254.1
                Communities: target:1234:1 encapsulation:vxlan(0x8)
                Import Accepted
                Route Label: 10010
                ESI: 00:00:00:00:00:00:00:00:00:00
                Localpref: 100
                Router ID: 10.255.255.2
                Primary Routing Table bgp.evpn.0

2:10.255.254.1:1::0::aa:bb:cc:00:0b:00::10.5.5.10/304 MAC/IP (2 entries, 1 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 10.255.254.1:1
                Next hop type: Indirect, Next hop index: 0
                Address: 0xce32a70
                Next-hop reference count: 12
                Source: 10.255.255.1
                Protocol next hop: 10.255.254.1
                Indirect next hop: 0x2 no-forward INH Session ID: 0x0
                State: <Secondary Active Int Ext>
                Local AS: 4210000001 Peer AS: 4210000001
                Age: 7:28       Metric2: 0
                Validation State: unverified
                Task: BGP_4210000001.10.255.255.1
                Announcement bits (1): 0-EVPN_10-evpn
                AS path: I  (Originator)
                Cluster list:  10.255.250.1
                Originator ID: 10.255.254.1
                Communities: target:1234:1 encapsulation:vxlan(0x8)
                Import Accepted
                Route Label: 10010
                ESI: 00:00:00:00:00:00:00:00:00:00
                Localpref: 100
                Router ID: 10.255.255.1
                Primary Routing Table bgp.evpn.0
         BGP    Preference: 170/-101
                Route Distinguisher: 10.255.254.1:1
                Next hop type: Indirect, Next hop index: 0
                Address: 0xce32a70
                Next-hop reference count: 12
                Source: 10.255.255.2
                Protocol next hop: 10.255.254.1
                Indirect next hop: 0x2 no-forward INH Session ID: 0x0
                State: <Secondary NotBest Int Ext Changed>
                Inactive reason: Not Best in its group - Update source
                Local AS: 4210000001 Peer AS: 4210000001
                Age: 7:28       Metric2: 0
                Validation State: unverified
                Task: BGP_4210000001.10.255.255.2
                AS path: I  (Originator)
                Cluster list:  10.255.250.1
                Originator ID: 10.255.254.1
                Communities: target:1234:1 encapsulation:vxlan(0x8)
                Import Accepted
                Route Label: 10010
                ESI: 00:00:00:00:00:00:00:00:00:00
                Localpref: 100
                Router ID: 10.255.255.2
                Primary Routing Table bgp.evpn.0
```

```
root@leaf-3> show route protocol bgp    
inet.0: 9 destinations, 11 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.255.254.1/32    *[BGP/170] 00:01:11, localpref 100, from 10.0.3.2
                      AS path: 65501 65511 I, validation-state: unverified
                      to 10.0.3.2 via xe-0/0/0.0
                    > to 10.0.3.6 via xe-0/0/1.0
                    [BGP/170] 00:01:11, localpref 100
                      AS path: 65501 65511 I, validation-state: unverified
                    > to 10.0.3.6 via xe-0/0/1.0
10.255.254.2/32    *[BGP/170] 00:01:12, localpref 100, from 10.0.3.2
                      AS path: 65501 65512 I, validation-state: unverified
                      to 10.0.3.2 via xe-0/0/0.0
                    > to 10.0.3.6 via xe-0/0/1.0
                    [BGP/170] 00:01:12, localpref 100
                      AS path: 65501 65512 I, validation-state: unverified
                    > to 10.0.3.6 via xe-0/0/1.0
```

Доступность Leaf-1, Leaf-2, Spine-1, Spine-2 с Leaf-3
```
{master:0}
root@leaf-3> ping 10.255.254.1 source 10.255.254.3 count 1
PING 10.255.254.1 (10.255.254.1): 56 data bytes
64 bytes from 10.255.254.1: icmp_seq=0 ttl=63 time=296.561 ms

--- 10.255.254.1 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 296.561/296.561/296.561/0.000 ms

{master:0}
root@leaf-3> ping 10.255.254.2 source 10.255.254.3 count 1
PING 10.255.254.2 (10.255.254.2): 56 data bytes
64 bytes from 10.255.254.2: icmp_seq=0 ttl=63 time=311.989 ms

--- 10.255.254.2 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 311.989/311.989/311.989/0.000 ms
```

Multipath работает
```
{master:0}
root@leaf-3> show route 10.255.254.1 detail 

inet.0: 9 destinations, 11 routes (9 active, 0 holddown, 0 hidden)
10.255.254.1/32 (2 entries, 1 announced)
        *BGP    Preference: 170/-101
                Next hop type: Router, Next hop index: 0
                Address: 0xb21bb10
                Next-hop reference count: 3
                Source: 10.0.3.2
                Next hop: 10.0.3.2 via xe-0/0/0.0
                Session Id: 0x0
                Next hop: 10.0.3.6 via xe-0/0/1.0, selected
                Session Id: 0x0
                State: <Active Ext>
                Peer AS: 65501
                Age: 2:18
                Validation State: unverified
                Task: BGP_65501_65513.10.0.3.2
                Announcement bits (2): 0-KRT 1-BGP_Listen.0.0.0.0+179
                AS path: 65501 65511 I
                Accepted Multipath
                Localpref: 100
                Router ID: 10.255.255.1
         BGP    Preference: 170/-101
                Next hop type: Router, Next hop index: 1740
                Address: 0xb3a1630
                Next-hop reference count: 3
                Source: 10.0.3.6
                Next hop: 10.0.3.6 via xe-0/0/1.0, selected
                Session Id: 0x0
                State: <NotBest Ext>
                Inactive reason: Not Best in its group - Active preferred
                Peer AS: 65501
                Age: 2:18
                Validation State: unverified
                Task: BGP_65501_65513.10.0.3.6
                AS path: 65501 65511 I
                Accepted MultipathContrib
                Localpref: 100
                Router ID: 10.255.255.2
```

```
R1#traceroute 31.173.128.1 numeric
Type escape sequence to abort.
Tracing the route to 31.173.128.1
VRF info: (vrf in name/id, vrf out name/id)
  1 10.5.5.100 4 msec 2 msec 8 msec
  2 10.5.5.200 7 msec 5 msec 5 msec
  3 31.173.128.1 10 msec 8 msec 6 msec
  
R1#ping 31.173.128.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 31.173.128.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 7/8/10 ms
R1#
```



```
R2#traceroute 31.173.128.1 numeric
Type escape sequence to abort.
Tracing the route to 31.173.128.1
VRF info: (vrf in name/id, vrf out name/id)
  1 172.17.0.100 3 msec 1 msec 2 msec
  2 10.5.5.200 7 msec 3 msec 6 msec
  3 31.173.128.1 7 msec 14 msec 11 msec

R2#ping 31.173.128.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 31.173.128.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 6/7/9 ms
```

```
R3#traceroute 31.173.128.1 numeric
Type escape sequence to abort.
Tracing the route to 31.173.128.1
VRF info: (vrf in name/id, vrf out name/id)
  1 10.5.5.200 2 msec 2 msec 2 msec
  2 31.173.128.1 5 msec 4 msec 4 msec

R3#ping 31.173.128.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 31.173.128.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/3/5 ms
R3#
```



## Итого

Сеть обеспечивает связность между клиентами по разным сервисам с должным резервированием.
Обновление статуса VTEP коммутаторов и туннелей происходит динамически.
В случае расширения сети новым подом можно использовать Juniper Data Center Interconnect.
