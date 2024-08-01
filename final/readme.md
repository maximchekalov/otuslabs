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

Для связности undelay будет применен протокол eBGP, для overlay связаности VxVlan.

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

Смоделирована сеть для ЦОД дизайном CLOS с применением модных техногий в виде EVPN VXVLAN. Применено резервирование MLAG.
