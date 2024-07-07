Домашнее задание №7

VXLAN. Multihoming.

Цель:
1. Настроить отказоустойчивое подключение клиентов с использованием EVPN Multihoming.
2. Подключить клиентов 2-я линками к различным Leaf.
3. Настроить агрегированный канал со стороны клиента.
4. Настроить multihoming для работы в Overlay сети.
5. Убедиться, что связность не теряется при отключении одного из линков.

Выполнение:
Схема сети

![image_CLOS_BGP_EVPN L3_MH](https://github.com/maximchekalov/otuslabs/blob/main/LABA7/MLAG.PNG)

Проверка доступности

![image](https://github.com/maximchekalov/otuslabs/blob/main/LABA7/leaf1mlag.PNG)
![image](https://github.com/maximchekalov/otuslabs/blob/main/LABA7/leaf1evpn.PNG)
![image](https://github.com/maximchekalov/otuslabs/blob/main/LABA7/leaf1vxvlan.PNG)
