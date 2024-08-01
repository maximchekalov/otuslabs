# Проектная работа

## Цель:

- Моделирование сети ЦОД на оборудовании Arista.

## План

- Разработка топологии
- Проектирование адресного пространства
- Проектирование сети Underlay
- Проектирование сети Overlay
- Проектирование клиентских подключений
- Реализация

## Разработка топологии
При разработке топологии сети важно учитывать объем трафика, который будет проходить через нее. В современных центрах обработки данных (DC) преобладает трафик направления "восток-запад" (East-West) по сравнению с направлением "север-юг" (North-South). Сетевая архитектура должна соответствовать следующим ключевым требованиям:
- масштабируемость
- отказоустойчивость
- простота эксплуатации
- отсутствие переподписки

Сеть для DC будеть иметь дизан CLOS, в которой имеются leaf и spine коммутаторы.

Leaf - оборудование доступа, куда включаются конечные потребители сети.

Spine - ядро сети, через которое проходит весь трафик между коммутаторами Leaf. Они не подключаются напрямую к серверам, а служат точками маршрутизации (или коммутации) для трафика между различными Leaf. К

![Пример сети CLOS](https://github.com/maximchekalov/otuslabs/blob/main/LABA1/topo.PNG)

Для связности undelay будет применен протокол eBGP, для overlay технологии EVPN VXVLAN.

## Проектирование адресного пространства

Распределение адресов для Loopback и p2p интерфейсов

| hostname | interface |   IP/MASK   | Description |
| :------: | :-------: | :----------: | :---------: |
|  leaf-1  | Loopback2 | 192.2.0.1 /32 |            |
|  leaf-1  |  eth 1/1  | 192.4.1.1 /31 | to-spine-1 |
|  leaf-1  |  eth 1/2  | 192.4.2.1 /31 | to-spine-2 |
|          |          |              |            |
|  leaf-2  | Loopback2 | 192.2.0.2 /32 |            |
|  leaf-2  |  eth 1/1  | 192.4.1.3 /31 | to-spine-1 |
|  leaf-2  |  eth 1/2  | 192.4.2.3 /31 | to-spine-2 |
|          |          |              |            |
|  leaf-3  | Loopback2 | 192.2.0.3 /32 |            |
|  leaf-3  |  eth 1/1  | 192.4.1.5 /31 | to-spine-1 |
|  leaf-3  |  eth 1/2  | 192.4.2.5 /31 | to-spine-2 |
|          |          |              |            |
| spine-1 | Loopback1 | 192.1.1.0/32 |            |
| spine-1 |  eth 1/1  | 192.4.1.0/31 |  to-leaf-1  |
| spine-1 |  eth 1/2  | 192.4.1.2/31 |  to-leaf-2  |
| spine-1 |  eth 1/3  | 192.4.1.4/31 |  to-leaf-3  |
|          |          |              |            |
| spine-2 | Loopback1 | 192.1.2.0/32 |            |
| spine-2 |  eth 1/1  | 192.4.2.0/31 |  to-leaf-1  |
| spine-2 |  eth 1/2  | 192.4.2.2/31 |  to-leaf-2  |
| spine-2 |  eth 1/3  | 192.4.2.2/31 |  to-leaf-3  |



## Проектирование сети Underlay
BGP удобен в использовании сетях DC, т.к можно гибко управлять трафиком, так же используя дополнительные фишки такие как ECMP для балансировки, MP-BGP для переноса сигнальной информации других протоколов.

eBGP работет по p2p и переносит только инфу о lo адресах.

## Проектирование сети Overlay
Оверлей будет работать при помощи протокола BGP и его функции evpn address family.

## Проектирование клиентских подключений
На проектируемой сети будут предоставлены услуги L2VPN, L3VPN
Резервирование обеспечивается на основе ESI LAG т.е. когда клиент одновременно включается в два и более leaf свитчей и 
организует LAG со свой стороны.


## Реализация

#### Пример конфигурации Underlay

leaf-1
```
hostname leaf1
!
spanning-tree mode mstp
no spanning-tree vlan-id 4094
!
vlan 100,200
!
vlan 4094
   trunk group mlag
!
vrf instance OTUSLABS
!
interface Port-Channel1
   description peerlink
   switchport mode trunk
   switchport trunk group mlag
!
interface Port-Channel2
   description to_srv_mlag
   switchport trunk allowed vlan 100,200
   switchport mode trunk
   mlag 2
!
interface Ethernet1
   description to-spine-1
   no switchport
   ip address 192.4.1.1/31
   bfd interval 100 min-rx 100 multiplier 3
!
interface Ethernet2
   description to-spine-2
   no switchport
   ip address 192.4.2.1/31
   bfd interval 100 min-rx 100 multiplier 3
!
interface Ethernet3
   description to_leaf_2
   channel-group 1 mode active
!
interface Ethernet4
   description to_SRV
   channel-group 2 mode active
!
interface Ethernet5
   description to_leaf2
   channel-group 1 mode active
!
interface Loopback2
   ip address 192.1.0.1/32
!
interface Loopback100
   description NVE_Loopback
   ip address 192.100.0.1/32
!
interface Management1
!
interface Vlan100
   vrf OTUSLABS
   ip address virtual 10.10.2.254/24
!
interface Vlan200
   vrf OTUSLABS
   ip address virtual 10.10.3.253/24
!
interface Vlan4094
   ip address 10.10.100.0/31
!
interface Vxlan1
   vxlan source-interface Loopback100
   vxlan udp-port 4789
   vxlan vlan 100 vni 100100
   vxlan vlan 200 vni 100200
   vxlan vrf OTUSLABS vni 999
   vxlan virtual-vtep local-interface Loopback100
!
ip virtual-router mac-address 00:00:00:00:00:01
!
ip routing
ip routing vrf OTUSLABS
!
ip prefix-list PL_LOOP
   seq 10 permit 192.1.0.1/32
   seq 20 permit 192.100.0.1/32
!
mlag configuration
   domain-id mlag1
   local-interface Vlan4094
   peer-address 10.10.100.1
   peer-link Port-Channel1
!
route-map RM_CONN permit 10
   match ip address prefix-list PL_LOOP
!
router bgp 65001
   router-id 192.1.0.1
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   neighbor EVPN peer group
   neighbor EVPN remote-as 65000
   neighbor EVPN update-source Loopback2
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor SPINE peer group
   neighbor SPINE remote-as 65000
   neighbor SPINE bfd
   neighbor SPINE allowas-in 1
   neighbor SPINE rib-in pre-policy retain all
   neighbor SPINE password 7 9cdhNWhDdTM=
   neighbor SPINE send-community
   neighbor SPINE maximum-routes 1000
   neighbor 192.1.1.0 peer group EVPN
   neighbor 192.1.2.0 peer group EVPN
   neighbor 192.4.1.0 peer group SPINE
   neighbor 192.4.2.0 peer group SPINE
   redistribute connected route-map RM_CONN
   !
   vlan 100
      rd 65001:100100
      route-target both 100:100
      redistribute learned
   !
   vlan 200
      rd 65001:100200
      route-target both 200:200
      redistribute learned
   !
   vlan 666
      rd 65001:10666
      route-target both 666:666
      redistribute learned
   !
   address-family evpn
      neighbor EVPN activate
   !
   vrf OTUSLABS
      rd 65001:999
      route-target import evpn 999:999
      route-target export evpn 999:999
      redistribute connected
!
end
```
spine-1
```
hostname spine-1
!
spanning-tree mode mstp
!
interface Ethernet1
   description to-leaf-1
   no switchport
   ip address 192.4.1.0/31
   bfd interval 100 min-rx 100 multiplier 3
!
interface Ethernet2
   description to-leaf-2
   no switchport
   ip address 192.4.1.2/31
   bfd interval 100 min-rx 100 multiplier 3
!
interface Ethernet3
   description to-leaf-3
   no switchport
   ip address 192.4.1.4/31
   bfd interval 100 min-rx 100 multiplier 3
!
interface Loopback1
   ip address 192.1.1.0/32
   ip ospf area 0.0.0.0
   isis enable underlay
!
interface Management1
!
ip routing
!
ip prefix-list PL_LOOP
   seq 10 permit 192.1.1.0/32
!
route-map RM_CONN permit 10
   match ip address prefix-list PL_LOOP
!
peer-filter EVPN
   10 match as-range 65001-65003 result accept
!
peer-filter LEAF
   10 match as-range 65001-65003 result accept
!
router bgp 65000
   router-id 192.1.1.0
   timers bgp 3 9
   maximum-paths 2 ecmp 2
   bgp listen range 192.1.0.0/24 peer-group EVPN peer-filter EVPN
   bgp listen range 192.4.1.0/24 peer-group LEAF peer-filter LEAF
   neighbor EVPN peer group
   neighbor EVPN next-hop-unchanged
   neighbor EVPN update-source Loopback1
   neighbor EVPN ebgp-multihop 3
   neighbor EVPN send-community extended
   neighbor LEAF peer group
   neighbor LEAF bfd
   neighbor LEAF rib-in pre-policy retain all
   neighbor LEAF password 7 7zCpOlU0aME=
   neighbor LEAF send-community
   neighbor LEAF maximum-routes 1000
   redistribute connected route-map RM_CONN
   !
   address-family evpn
      neighbor EVPN activate
!
end
```



### Топология сети 

![image](https://github.com/maximchekalov/otuslabs/blob/main/final/CLOS%20Final.PNG)


### Вывод диангостической информации 
Leaf1
```
leaf1#sh ip bgp summary
BGP summary information for VRF default
Router identifier 192.1.0.1, local AS number 65001
Neighbor Status Codes: m - Under maintenance
  Neighbor  V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  192.1.1.0 4 65000           3117      3101    0    0 01:49:57 Estab   5      5
  192.1.2.0 4 65000           3065      3136    0    0 01:49:58 Estab   5      5
  192.4.1.0 4 65000           2737      2770    0    0 00:03:02 Estab   5      5
  192.4.2.0 4 65000           2715      2751    0    0 00:03:02 Estab   5      5

```

Spine1
```
spine-1#sh ip bgp summary
BGP summary information for VRF default
Router identifier 192.1.1.0, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Neighbor  V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  192.1.0.1 4 65001           3115      3131    0    0 01:50:34 Estab   2      2
  192.1.0.2 4 65002           3099      3094    0    0 01:50:31 Estab   2      2
  192.1.0.3 4 65003           2974      3034    0    0 01:50:31 Estab   2      2
  192.4.1.1 4 65001             90        91    0    0 00:03:39 Estab   3      2
  192.4.1.3 4 65002             90        91    0    0 00:03:38 Estab   3      2
  192.4.1.5 4 65003             90        91    0    0 00:03:38 Estab   3      2
```




```
leaf1#sh bgp evpn vni 999
BGP routing table information for VRF default
Router identifier 192.1.0.1, local AS number 65001
Route status codes: s - suppressed, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec   RD: 65003:100100 mac-ip 0050.7966.6808 10.10.2.11
                                 192.100.0.3           -       100     0       65000 65003 i
 *  ec   RD: 65003:100100 mac-ip 0050.7966.6808 10.10.2.11
                                 192.100.0.3           -       100     0       65000 65003 i
 * >     RD: 65001:100100 mac-ip 5018.00c9.18c1 10.10.2.10
                                 -                     -       -       0       i
 * >Ec   RD: 65003:999 ip-prefix 0.0.0.0/0
                                 192.100.0.3           -       100     0       65000 65003 64009 ?
 *  ec   RD: 65003:999 ip-prefix 0.0.0.0/0
                                 192.100.0.3           -       100     0       65000 65003 64009 ?
 * >Ec   RD: 65003:999 ip-prefix 8.8.4.4/32
                                 192.100.0.3           -       100     0       65000 65003 64009 i
 *  ec   RD: 65003:999 ip-prefix 8.8.4.4/32
                                 192.100.0.3           -       100     0       65000 65003 64009 i
 * >Ec   RD: 65003:999 ip-prefix 8.8.8.8/32
                                 192.100.0.3           -       100     0       65000 65003 64009 i
 *  ec   RD: 65003:999 ip-prefix 8.8.8.8/32
                                 192.100.0.3           -       100     0       65000 65003 64009 i
 * >     RD: 65001:999 ip-prefix 10.10.2.0/24
                                 -                     -       -       0       i
 * >Ec   RD: 65002:999 ip-prefix 10.10.2.0/24
                                 192.100.0.2           -       100     0       65000 65002 i
 *  ec   RD: 65002:999 ip-prefix 10.10.2.0/24
                                 192.100.0.2           -       100     0       65000 65002 i
 * >Ec   RD: 65003:999 ip-prefix 10.10.2.0/24
                                 192.100.0.3           -       100     0       65000 65003 i
 *  ec   RD: 65003:999 ip-prefix 10.10.2.0/24
                                 192.100.0.3           -       100     0       65000 65003 i
 * >     RD: 65001:999 ip-prefix 10.10.3.0/24
                                 -                     -       -       0       i
 * >Ec   RD: 65003:999 ip-prefix 10.10.3.0/24
                                 192.100.0.3           -       100     0       65000 65003 i
 *  ec   RD: 65003:999 ip-prefix 10.10.3.0/24
                                 192.100.0.3           -       100     0       65000 65003 i
 * >Ec   RD: 65003:999 ip-prefix 10.200.0.1/32
                                 192.100.0.3           -       100     0       65000 65003 64009 i
 *  ec   RD: 65003:999 ip-prefix 10.200.0.1/32
                                 192.100.0.3           -       100     0       65000 65003 64009 i
 * >Ec   RD: 65003:999 ip-prefix 33.33.33.0/24
                                 192.100.0.3           -       100     0       65000 65003 64009 i
 *  ec   RD: 65003:999 ip-prefix 33.33.33.0/24
                                 192.100.0.3           -       100     0       65000 65003 64009 i
 * >Ec   RD: 65003:999 ip-prefix 66.66.66.0/24
                                 192.100.0.3           -       100     0       65000 65003 i
 *  ec   RD: 65003:999 ip-prefix 66.66.66.0/24
                                 192.100.0.3           -       100     0       65000 65003 i
```

### Связанность клиентов
VPC8 - SRV - INTERNET 
```
VPCS> ping 10.10.2.10

84 bytes from 10.10.2.10 icmp_seq=1 ttl=64 time=12.974 ms
84 bytes from 10.10.2.10 icmp_seq=2 ttl=64 time=14.298 ms
84 bytes from 10.10.2.10 icmp_seq=3 ttl=64 time=15.029 ms
^C
VPCS> ping 8.8.8.8

84 bytes from 8.8.8.8 icmp_seq=1 ttl=63 time=8.284 ms
84 bytes from 8.8.8.8 icmp_seq=2 ttl=63 time=7.422 ms
84 bytes from 8.8.8.8 icmp_seq=3 ttl=63 time=7.204 ms
^C
VPCS>

```
VPC9 - VPC8 - SRV - INTERNET
```
VPCS> ping 10.10.3.10

10.10.3.10 icmp_seq=1 timeout
84 bytes from 10.10.3.10 icmp_seq=2 ttl=64 time=13.801 ms
84 bytes from 10.10.3.10 icmp_seq=3 ttl=64 time=19.272 ms
84 bytes from 10.10.3.10 icmp_seq=4 ttl=64 time=17.746 ms
^C
VPCS> ping 10.10.2.11

84 bytes from 10.10.2.11 icmp_seq=1 ttl=63 time=5.962 ms
84 bytes from 10.10.2.11 icmp_seq=2 ttl=63 time=5.783 ms
84 bytes from 10.10.2.11 icmp_seq=3 ttl=63 time=5.597 ms
^C
VPCS> ping 8.8.4.4

84 bytes from 8.8.4.4 icmp_seq=1 ttl=63 time=7.159 ms
84 bytes from 8.8.4.4 icmp_seq=2 ttl=63 time=8.395 ms
84 bytes from 8.8.4.4 icmp_seq=3 ttl=63 time=8.937 ms
^C
VPCS>
```
Проверка связанности сетевого оборудования через Lo
```
leaf1#ping 192.1.0.2 source 192.1.0.1
PING 192.1.0.2 (192.1.0.2) from 192.1.0.1 : 72(100) bytes of data.
80 bytes from 192.1.0.2: icmp_seq=1 ttl=63 time=6.17 ms
80 bytes from 192.1.0.2: icmp_seq=2 ttl=63 time=5.48 ms
80 bytes from 192.1.0.2: icmp_seq=3 ttl=63 time=5.70 ms
80 bytes from 192.1.0.2: icmp_seq=4 ttl=63 time=5.61 ms
80 bytes from 192.1.0.2: icmp_seq=5 ttl=63 time=5.65 ms

--- 192.1.0.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 31ms
rtt min/avg/max/mdev = 5.482/5.726/6.178/0.242 ms, ipg/ewma 7.880/5.947 ms
leaf1#
leaf1#ping 192.1.0.3 ource 192.1.0.1
PING 192.1.0.3 (192.1.0.3) from 192.1.0.1 : 72(100) bytes of data.
80 bytes from 192.1.0.3: icmp_seq=1 ttl=63 time=6.34 ms
80 bytes from 192.1.0.3: icmp_seq=2 ttl=63 time=6.26 ms
80 bytes from 192.1.0.3: icmp_seq=3 ttl=63 time=6.42 ms
80 bytes from 192.1.0.3: icmp_seq=4 ttl=63 time=5.70 ms
80 bytes from 192.1.0.3: icmp_seq=5 ttl=63 time=5.32 ms

--- 192.1.0.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 31ms
rtt min/avg/max/mdev = 5.320/6.010/6.423/0.439 ms, ipg/ewma 7.796/6.147 ms

leaf1#
leaf1#ping 192.1.2. source 192.1.0.1
PING 192.1.2.0 (192.1.2.0) from 192.1.0.1 : 72(100) bytes of data.
80 bytes from 192.1.2.0: icmp_seq=1 ttl=64 time=2.91 ms
80 bytes from 192.1.2.0: icmp_seq=2 ttl=64 time=2.17 ms
80 bytes from 192.1.2.0: icmp_seq=3 ttl=64 time=1.91 ms
80 bytes from 192.1.2.0: icmp_seq=4 ttl=64 time=3.54 ms
80 bytes from 192.1.2.0: icmp_seq=5 ttl=64 time=1.99 ms

--- 192.1.2.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 15ms
rtt min/avg/max/mdev = 1.913/2.508/3.546/0.627 ms, ipg/ewma 3.943/2.709 ms
leaf1#
leaf1#ping 192.1.1. source 192.1.0.1
PING 192.1.1.0 (192.1.1.0) from 192.1.0.1 : 72(100) bytes of data.
80 bytes from 192.1.1.0: icmp_seq=1 ttl=64 time=2.86 ms
80 bytes from 192.1.1.0: icmp_seq=2 ttl=64 time=2.27 ms
80 bytes from 192.1.1.0: icmp_seq=3 ttl=64 time=2.09 ms
80 bytes from 192.1.1.0: icmp_seq=4 ttl=64 time=4.29 ms
80 bytes from 192.1.1.0: icmp_seq=5 ttl=64 time=2.30 ms

--- 192.1.1.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 19ms
rtt min/avg/max/mdev = 2.094/2.766/4.293/0.808 ms, ipg/ewma 4.858/2.829 ms
```


Проверка MLAG
```
leaf1# sh mlag
MLAG Configuration:
domain-id                          :               mlag1
local-interface                    :            Vlan4094
peer-address                       :         10.10.100.1
peer-link                          :       Port-Channel1
peer-config                        :        inconsistent

MLAG Status:
state                              :              Active
negotiation status                 :           Connected
peer-link status                   :                  Up
local-int status                   :                  Up
system-id                          :   52:18:00:27:35:a8
dual-primary detection             :            Disabled
dual-primary interface errdisabled :               False

MLAG Ports:
Disabled                           :                   0
Configured                         :                   0
Inactive                           :               0
Active-partial                     :                   0
Active-full                        :                   1

```


## Вывод

Смоделирована сеть для ЦОД дизайном CLOS с применением модных техногий в виде EVPN VXVLAN. Применено резервирование MLAG.
