# otuslabs
# Домашнее задание №1
## Проектирование адресного пространства

### Задачи:

- Собрать схему CLOS;
- Распределить адресное пространство.

## Выполнение:

### Собранная схема сети

![Иллюстрация к проекту](https://github.com/maximchekalov/otuslabs/blob/main/LABA1/topo.PNG) 


### Таблица адресов

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

### Конфигурация оборудования

- #### [leaf-1](config/leaf-1.conf)

```
hostname leaf-1

interface Ethernet1/1
  description to-spine-1
  no switchport
  ip address 192.4.1.1/31
  no shutdown

interface Ethernet1/2
  description to-spine-2
  no switchport
  ip address 192.4.2.1/31
  no shutdown
  
interface loopback2
  ip address 192.1.0.1/32
```

- #### [leaf-2](config/leaf-2.conf)

```
hostname leaf-2

interface Ethernet1/1
  description to-spine-1
  no switchport
  ip address 192.4.1.3/31
  no shutdown

interface Ethernet1/2
  description to-spine-2
  no switchport
  ip address 192.4.2.3/31
  no shutdown
  
interface loopback2
  ip address 192.1.0.2/32
```

- #### [leaf-3](config/leaf-3.conf)

```
hostname leaf-3

interface Ethernet1/1
  description to-spine-1
  no switchport
  ip address 192.4.1.5/31
  no shutdown

interface Ethernet1/2
  description to-spine-2
  no switchport
  ip address 192.4.2.5/31
  no shutdown
  
interface loopback2
  ip address 192.1.0.3/32
```

- #### [spine-1](config/spine-1.conf)

```
hostname spine-1

interface Ethernet1/1
  description to-leaf-1
  no switchport
  ip address 192.4.1.0/31
  no shutdown

interface Ethernet1/2
  description to-leaf-2
  no switchport
  ip address 192.4.1.2/31
  no shutdown

interface Ethernet1/3
  description to-leaf-3
  no switchport
  ip address 192.4.1.4/31
  no shutdown

interface loopback1
  ip address 192.1.1.0/32
```

- #### [spine-2](config/spine-2.conf)

```
hostname spine-2

interface Ethernet1/1
  description to-leaf-1
  no switchport
  ip address 192.4.2.0/31
  no shutdown

interface Ethernet1/2
  description to-leaf-2
  no switchport
  ip address 192.4.2.2/31
  no shutdown

interface Ethernet1/3
  description to-leaf-3
  no switchport
  ip address 192.4.2.4/31
  no shutdown
  
interface loopback1
  ip address 192.1.2.0/32
```

### Проверка доступности
![Иллюстрация к проекту](https://github.com/maximchekalov/otuslabs/blob/main/LABA1/spine2.PNG)
![Иллюстрация к проекту](https://github.com/maximchekalov/otuslabs/blob/main/LABA1/spine1.PNG)
