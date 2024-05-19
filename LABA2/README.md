# Домашнее задание №2
## Underlay. OSPF

### Задача:

- Настроить протокол OSPF для Underlay сети
- Проверить связанность между устройствами

## Выполнение:
## Spine2

```
hostname spine-2
!
spanning-tree mode mstp
!
interface Ethernet1
   description to-leaf-1
   no switchport
   ip address 192.4.2.0/31
   bfd interval 100 min_rx 100 multiplier 3
   ip ospf neighbor bfd
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 21 md5 7 ItKol34eEhFU8Zs/V+hF3g==
!
interface Ethernet2
   description to-leaf-2
   no switchport
   ip address 192.4.2.2/31
   bfd interval 100 min_rx 100 multiplier 3
   ip ospf neighbor bfd
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 22 md5 7 sBDK09qRIOTpSfj2f4+8HQ==
!
interface Ethernet3
   description to-leaf-3
   no switchport
   ip address 192.4.2.4/31
   bfd interval 100 min_rx 100 multiplier 3
   ip ospf neighbor bfd
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 23 md5 7 sBDK09qRIOTobg6bWrwctg==
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback1
   ip address 192.1.2.0/32
   ip ospf area 0.0.0.0
!
interface Management1
!
ip routing
!
router ospf 1
   router-id 192.1.2.0
   bfd default
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   max-lsa 12000
   log-adjacency-changes detail
!
end
```
##Spine1
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
   ip ospf neighbor bfd
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 11 md5 7 yDPXuPQG09LXmDBgF5D88g==
!
interface Ethernet2
   description to-leaf-2
   no switchport
   ip address 192.4.1.2/31
   bfd interval 100 min-rx 100 multiplier 3
   ip ospf neighbor bfd
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 12 md5 7 ISVBJOG/W5ChlltrslFnGw==
!
interface Ethernet3
   description to-leaf-3
   no switchport
   ip address 192.4.1.4/31
   bfd interval 100 min-rx 100 multiplier 3
   ip ospf neighbor bfd
   ip ospf network point-to-point
   ip ospf authentication message-digest
   ip ospf area 0.0.0.0
   ip ospf message-digest-key 13 md5 7 ISVBJOG/W5BaLOZtaRbphw==
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback1
   ip address 192.1.1.0/32
   ip ospf area 0.0.0.0
!
interface Management1
!
ip routing
!
router ospf 1
   router-id 192.1.1.0
   bfd default
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   max-lsa 12000
   log-adjacency-changes detail
!
end
```

### Схема сети
![Иллюстрация к проекту](https://github.com/maximchekalov/otuslabs/blob/main/LABA1/topo.PNG) 

![Иллюстрация к проекту](https://github.com/maximchekalov/otuslabs/blob/main/LABA2/spine1ospf.png)
![Иллюстрация к проекту](https://github.com/maximchekalov/otuslabs/blob/main/LABA2/spine2ospf.png)
