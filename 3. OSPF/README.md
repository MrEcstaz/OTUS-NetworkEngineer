# OSPF: многозонная конфигурация

![Топология](topology.png)

Офис Москва (AS 1001): R12 и R13 с пользовательскими VLAN (настроены в [работе 1](../1.%20Проектирование%20сети/README.md)), ABR R14 и R15, периферийные R19 и R20, коммутаторы SW2, SW3, SW4, SW5 и пинг-машины VPC1 (VLAN 10) и VPC7 (VLAN 20). Линк R14 e0/2 (10.0.100.0/30) уходит к R22, линк R15 e0/2 (10.0.100.12/30) к R21: оба в другие AS и в OSPF не участвуют.

## 1. Задача и допущения

Требования задания:

1. R14 и R15 в зоне 0 (backbone).
2. R12 и R13 в зоне 10, дополнительно получают маршрут по умолчанию.
3. R19 в зоне 101, получает только маршрут по умолчанию.
4. R20 в зоне 102, получает все маршруты, кроме сетей зоны 101.
5. Логика повторяется для IPv6.
6. План и изменения зафиксированы в документации.

Решения, которые я принял до работы со стендом:

- Зона 0 логически разорвана: R14 и R15 не соединены напрямую, все их порты заняты, физический линк добавить нельзя. В зоне 0 лежат только Loopback0 у R14 (10.0.0.14/32) и R15 (10.0.0.15/32), adjacency между ABR нет. Что из этого выходит для маршрутизации, показал стенд, всё описано по факту в разделе 6.
- Virtual link я не использовал: тема вне программы занятия, и через stub-зону virtual link в принципе невозможен ([RFC 2328, раздел 3.6](https://www.rfc-editor.org/rfc/rfc2328#section-3.6)).
- IPv6 я не настраивал: пункт 5 задания выполняется частично, вся логика зон на IPv4. Это осознанное решение, помечаю его здесь и в разделе 9.
- Зона 10: stub. ABR (R14, R15) автоматически вставляют в stub-зону default 0.0.0.0/0, так что пункт 2 (маршрут по умолчанию для R12 и R13) закрывается без отдельной команды.
- Зона 101: totally stubby: `area 101 stub no-summary` на ABR R14, `area 101 stub` на R19. R19 получает только default, type 3 LSA в зону не проходят.
- Зона 102: обычная. Чтобы выполнить пункт 4, на ABR R15 стоит фильтр `area 102 filter-list prefix NO-AREA101 in`, который не выпускает в зону 102 сети зоны 101: 10.0.0.19/32 (loopback R19) и 10.0.1.8/30 (линк R14 R19).
- Суммаризация: `area 10 range 10.11.0.0 255.255.0.0` на R14 и R15. Три пользовательских /24 Москвы (10, 20, 99) сворачиваются в один /16 в LSA, уходящих из зоны 10.
- NSSA в конфигурацию не вошёл: внешних маршрутов в Москве нет, ASBR нет. Типы зон с NSSA я разберу в разделе 3, чтобы закрыть тему занятия.

| Зона | Тип | Роутеры | Зачем |
|---|---|---|---|
| 0 | backbone (разорвана) | R14, R15 | по заданию, ABR |
| 10 | stub | R12, R13, R14, R15 | офис, external маршруты не нужны, ABR вставит default |
| 101 | totally stubby | R19, R14 | максимум упрощения: только default |
| 102 | обычная + фильтр на R15 | R20, R15 | все маршруты, кроме сетей зоны 101 |

Интерфейсы, которые я включил в OSPF:

| Роутер | Интерфейс | Сеть | Зона |
|---|---|---|---|
| R12 | Loopback0 | 10.0.0.12/32 | 10 |
| R12 | e0/0.10 | 10.11.10.0/24 | 10 |
| R12 | e0/0.99 | 10.11.99.0/24 | 10 |
| R12 | e0/1.20 | 10.11.20.0/24 | 10 |
| R12 | e0/2 | 10.0.1.0/30 (к R14) | 10 |
| R12 | e0/3 | 10.0.1.16/30 (к R15) | 10 |
| R13 | Loopback0 | 10.0.0.13/32 | 10 |
| R13 | e0/0.10 | 10.11.10.0/24 | 10 |
| R13 | e0/0.99 | 10.11.99.0/24 | 10 |
| R13 | e0/1.20 | 10.11.20.0/24 | 10 |
| R13 | e0/3 | 10.0.1.4/30 (к R14) | 10 |
| R13 | e0/2 | 10.0.1.12/30 (к R15) | 10 |
| R14 | Loopback0 | 10.0.0.14/32 | 0 |
| R14 | e0/0 | 10.0.1.0/30 (к R12) | 10 |
| R14 | e0/1 | 10.0.1.4/30 (к R13) | 10 |
| R14 | e0/3 | 10.0.1.8/30 (к R19) | 101 |
| R15 | Loopback0 | 10.0.0.15/32 | 0 |
| R15 | e0/0 | 10.0.1.12/30 (к R13) | 10 |
| R15 | e0/1 | 10.0.1.16/30 (к R12) | 10 |
| R15 | e0/3 | 10.0.1.20/30 (к R20) | 102 |
| R19 | Loopback0 | 10.0.0.19/32 | 101 |
| R19 | e0/0 | 10.0.1.8/30 (к R14) | 101 |
| R20 | Loopback0 | 10.0.0.20/32 | 102 |
| R20 | e0/0 | 10.0.1.20/30 (к R15) | 102 |

R12 и R13 были настроены ещё в работе 1: IP на саб-интерфейсах, HSRP (vIP 10.11.10.1 у R12/R13, vIP 10.11.20.1), управление. В эту работу на них добавил только OSPF.

## 2. С чего я начал

Перед изменениями на R12, R14, R15, R19, R20 снял `show ip route` и `show ip ospf neighbor`. OSPF нигде не был настроен, у каждого роутера только connected-сети:

```text
R14#show ip route
    10.0.0.0/8 is variably subnetted, 9 subnets, 2 masks
C        10.0.0.14/32 is directly connected, Loopback0
C        10.0.1.0/30 is directly connected, Ethernet0/0
C        10.0.1.4/30 is directly connected, Ethernet0/1
C        10.0.1.8/30 is directly connected, Ethernet0/3
C        10.0.100.0/30 is directly connected, Ethernet0/2
```

```text
R19#show ip route
    10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
C        10.0.0.19/32 is directly connected, Loopback0
C        10.0.1.8/30 is directly connected, Ethernet0/0
```

Из этого я взял две отправные точки: какие connected-сети были до, и что ни один роутер не знает ни адресов VLAN, ни loopback'ов соседей.

## 3. План и порядок

Порядок я выстроил так, чтобы каждый этап можно было проверить до следующего:

1. Снять базовое состояние (раздел 2).
2. R12 и R13: зона 10, stub. Проверка: оба видят своих ABR.
3. R14 и R15: ABR, зоны 10, 101, 102. Проверка: `show ip ospf neighbor`, все adjacency FULL.
4. R19 (зона 101) и R20 (зона 102). Проверка: таблицы маршрутов, «до фильтра».
5. Суммаризация зоны 10: `area 10 range 10.11.0.0 255.255.0.0` на обоих ABR. Проверка: у R20 один /16 вместо трёх /24.
6. Фильтр на R15: prefix-list + `area 102 filter-list prefix NO-AREA101 in`. Проверка: таблицы R20 и R15, prefix-list.
7. Финальная матрица пингов (раздел 5).
8. `copy running-config startup-config` на всех шести роутерах.

По типам зон (тема занятия) для ориентира:

| Тип | External LSA (5) | Inter-area LSA (3) | Default от ABR | Когда применять |
|---|---|---|---|---|
| Обычная | да | да | нет | зоны, которым нужны все маршруты |
| Stub | нет | да | да | периметр сети, внешние маршруты не нужны |
| Totally stubby | нет | нет | да | максимум упрощения таблицы, только default |
| NSSA | да, но локальные (type 7) | да | да | зона со своим ASBR |
| Totally NSSA | да, но локальные (type 7) | нет | да | NSSA плюс подавление type 3 |

В этой работе использованы обычная (102), stub (10) и totally stubby (101).

## 4. Ход работы

### 4.1 R12 и R13: зона 10, stub

R12:

```text
router ospf 1
 log-adjacency-changes
 area 10 stub
 passive-interface default
 no passive-interface Ethernet0/2
 no passive-interface Ethernet0/3
!
interface Loopback0
 ip ospf 1 area 10
!
interface Ethernet0/0.10
 ip ospf 1 area 10
!
interface Ethernet0/0.99
 ip ospf 1 area 10
!
interface Ethernet0/1.20
 ip ospf 1 area 10
!
interface Ethernet0/2
 ip ospf 1 area 10
!
interface Ethernet0/3
 ip ospf 1 area 10
```

R13 то же самое, с перестановкой портов: e0/3 к R14, e0/2 к R15.

Почему так:

- `area 10 stub` нужен на каждом роутере зоны, включая ABR. Stub-зона не принимает external LSA (type 5), а ABR автоматически вставляют в неё default: это и есть маршрут по умолчанию для R12 и R13 из пункта 2 задания.
- `passive-interface default` + `no passive-interface` гасят hello на всех интерфейсах, кроме двух P2P-линков до ABR. Саб-интерфейсы к пользовательским VLAN остаются пассивными: hello в VLAN без роутеров не нужно.
- В IOS OSPF включается на интерфейсе явно командой `ip ospf 1 area N`, поэтому линки вне OSPF достаточно просто не настраивать.

### 4.2 R14 и R15: ABR

R14:

```text
router ospf 1
 log-adjacency-changes
 area 10 stub
 area 101 stub no-summary
!
interface Loopback0
 ip ospf 1 area 0
!
interface Ethernet0/0
 ip ospf 1 area 10
!
interface Ethernet0/1
 ip ospf 1 area 10
!
interface Ethernet0/3
 ip ospf 1 area 101
```

R15:

```text
router ospf 1
 log-adjacency-changes
 area 10 stub
!
interface Loopback0
 ip ospf 1 area 0
!
interface Ethernet0/0
 ip ospf 1 area 10
!
interface Ethernet0/1
 ip ospf 1 area 10
!
interface Ethernet0/3
 ip ospf 1 area 102
```

`no-summary` действует только на ABR, поэтому на R14 стоит `area 101 stub no-summary`, а R15 эту зону не знает. Линки e0/2 на обоих ABR (к R22 и R21, другие AS) в OSPF не включил.

Проверка adjacency на обоих ABR:

```text
R14#show ip ospf neighbor
Neighbor ID   Pri   State         Dead Time   Address       Interface
10.0.0.13     1     FULL/DR       00:00:32    10.0.1.5      Ethernet0/1
10.0.0.12     1     FULL/DR       00:00:39    10.0.1.1      Ethernet0/0
10.0.0.19     1     FULL/BDR      00:00:31    10.0.1.10     Ethernet0/3

R15#show ip ospf neighbor
Neighbor ID   Pri   State         Dead Time   Address       Interface
10.0.0.12     1     FULL/DR       00:00:36    10.0.1.17     Ethernet0/1
10.0.0.13     1     FULL/DR       00:00:34    10.0.1.13     Ethernet0/0
10.0.0.20     1     FULL/BDR      00:00:32    10.0.1.22     Ethernet0/3
```

Все adjacency FULL. Соседей в зоне 0 нет: ABR физически не соединены, это осознанное состояние, а не ошибка.

### 4.3 R19 (зона 101) и R20 (зона 102)

R19:

```text
router ospf 1
 log-adjacency-changes
 area 101 stub
!
interface Loopback0
 ip ospf 1 area 101
!
interface Ethernet0/0
 ip ospf 1 area 101
```

R20:

```text
router ospf 1
 log-adjacency-changes
!
interface Loopback0
 ip ospf 1 area 102
!
interface Ethernet0/0
 ip ospf 1 area 102
```

Таблица R19 после поднятия OSPF:

```text
R19#show ip route
Gateway of last resort is 10.0.1.9 to network 0.0.0.0

O*IA   0.0.0.0/0 [110/11] via 10.0.1.9, 00:06:10, Ethernet0/0
    10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
C        10.0.0.19/32 is directly connected, Loopback0
C        10.0.1.8/30 is directly connected, Ethernet0/0
```

Только connected-сети и default от ABR, межзонных маршрутов нет. totally stubby работает, пункт 3 задания закрыт.

Таблица R20 «до фильтра» (зона 102 обычная, R20 видит то, что R15 анонсирует):

```text
R20#show ip route
    10.0.0.0/8 is variably subnetted, 13 subnets, 3 masks
O IA   10.0.0.12/32 [110/21] via 10.0.1.21, 00:13:14, Ethernet0/0
O IA   10.0.0.13/32 [110/21] via 10.0.1.21, 00:13:14, Ethernet0/0
O IA   10.0.0.15/32 [110/11] via 10.0.1.21, 00:13:14, Ethernet0/0
C      10.0.0.20/32 is directly connected, Loopback0
O IA   10.0.1.0/30 [110/30] via 10.0.1.21, 00:13:14, Ethernet0/0
O IA   10.0.1.4/30 [110/30] via 10.0.1.21, 00:13:14, Ethernet0/0
O IA   10.0.1.12/30 [110/20] via 10.0.1.21, 00:13:14, Ethernet0/0
O IA   10.0.1.16/30 [110/20] via 10.0.1.21, 00:13:14, Ethernet0/0
C      10.0.1.20/30 is directly connected, Ethernet0/0
O IA   10.11.10.0/24 [110/30] via 10.0.1.21, 00:13:14, Ethernet0/0
O IA   10.11.20.0/24 [110/30] via 10.0.1.21, 00:13:14, Ethernet0/0
O IA   10.11.99.0/24 [110/30] via 10.0.1.21, 00:13:14, Ethernet0/0
```

Сразу отметился важная деталь: в таблице R20 НЕТ 10.0.0.19/32 и 10.0.1.8/30 (сети зоны 101), и при этом тоже нет 10.0.0.14/32 (loopback R14). Проверил, знает ли это R15:

```text
R15#show ip route
O*IA   0.0.0.0/0  [110/21] via 10.0.1.17
                    [110/21] via 10.0.1.13
O IA   10.0.0.12/32 [110/11] via 10.0.1.17
O IA   10.0.0.13/32 [110/11] via 10.0.1.13
O IA   10.0.0.14/32 [110/21] via 10.0.1.17
                    [110/21] via 10.0.1.13
C      10.0.0.15/32 is directly connected, Loopback0
O IA   10.0.0.19/32 [110/31] via 10.0.1.17
                    [110/31] via 10.0.1.13
O      10.0.0.20/32 [110/11] via 10.0.1.22
O IA   10.0.1.8/30  [110/30] via 10.0.1.17
                    [110/30] via 10.0.1.13
C      10.0.1.12/30 is directly connected, Ethernet0/0
C      10.0.1.16/30 is directly connected, Ethernet0/1
C      10.0.1.20/30 is directly connected, Ethernet0/3
O      10.11.10.0/24 [110/20] via 10.0.1.17
O      10.11.20.0/24 [110/20] via 10.0.1.17
O      10.11.99.0/24 [110/20] via 10.0.1.17
```

У R15 есть и 10.0.0.14/32, и 10.0.0.19/32, и 10.0.1.8/30, и R15 знает их через зону 10: R14 как ABR анонсирует свой loopback (зона 0) и сети зоны 101 в зону 10, R15 это получает. Но в зону 102 R15 эти префиксы не переносит. Вывод: на этом стенде фильтр из следующего раздела будет демонстрационным, сети зоны 101 R20 не увидит и без него. Зафиксировал это здесь, чтобы раздел 5 читался без сюрпризов.

### 4.4 Суммаризация зоны 10

```text
R14(config-router)#area 10 range 10.11.0.0 255.255.0.0
R15(config-router)#area 10 range 10.11.0.0 255.255.0.0
```

Команда создаёт суммарный префикс 10.11.0.0/16, которым ABR заменяет в type 3 LSA все /24 зоны 10, попадающие в диапазон. Внутри самой зоны 10 ничего не меняется.

Проверка на R20:

```text
R20#show ip route 10.11.0.0
Routing entry for 10.11.0.0/16
    Known via "ospf 1", distance 110, metric 30, type inter area
    Last update from 10.0.1.21 on Ethernet0/0
    * 10.0.1.21, from 10.0.0.15
```

Три /24 (10.11.10, 10.11.20, 10.11.99) в таблице R20 исчезли, вместо них один O IA 10.11.0.0/16 [110/30]. Таблица стала короче с 13 подсетей до 11.

Проверка внутри зоны:

```text
R12#show ip route 10.11.10.0
Routing entry for 10.11.10.0/24
    Known via "connected", via Ethernet0/0.10
```

Точные /24 внутри зоны 10 сохранились: суммаризация действует только на LSA, уходящие из зоны 10 во внешние зоны.

Побочный эффект, который я заметил сразу после range на R15:

```text
R15#show ip route 10.11.0.0
Routing entry for 10.11.0.0/16
    Known via "ospf 1", distance 110, metric 20, type intra area
    directly connected, via Null0
```

ABR получает локальный маршрут к суммарному префиксу с next hop Null0: трафик, попавший в /16, но не в ни один реальный /24, будет сброшен вместо ухода в затылок.

### 4.5 Фильтр зоны 101 на R15

```text
ip prefix-list NO-AREA101 seq 5 deny 10.0.0.19/32
ip prefix-list NO-AREA101 seq 10 deny 10.0.1.8/30
ip prefix-list NO-AREA101 seq 15 permit 0.0.0.0/0 le 32
!
router ospf 1
 area 102 filter-list prefix NO-AREA101 in
```

Первые две строки отклоняют сети зоны 101, третья пропускает остальное: без неё список отфильтровал бы всё, потому что неявное действие в конце prefix-list deny. Направление `in` на `area filter-list` означает фильтрацию type 3 LSA, которые R15 публикует в зону 102.

Проверки:

```text
R15#show ip prefix-list NO-AREA101
ip prefix-list NO-AREA101: 3 entries
    seq 5 deny 10.0.0.19/32
    seq 10 deny 10.0.1.8/30
    seq 15 permit 0.0.0.0/0 le 32

R15#show ip route 10.0.0.19
Routing entry for 10.0.0.19/32
    Known via "ospf 1", distance 110, metric 31, type inter area
    10.0.1.17, from 10.0.0.14, via Ethernet0/1
    10.0.1.13, from 10.0.0.14, via Ethernet0/0

R20#show ip route 10.0.0.19
% Subnet not in table
R20#show ip route 10.0.1.8
% Subnet not in table
```

Маршруты в собственной таблице R15 сохранились: фильтр не трогает локальную таблицу ABR. На R20 сетей зоны 101 нет: как и до фильтра (раздел 4.3), так и после. Как я это читаю: пункт 4 задания (R20 не получает сети зоны 101) на стенде выполнен, но не фильтром, а поведением ABR: R15 в зону 102 анонсирует только маршруты, которые он знает «своей» стороны, а маршруты, полученные им из зоны 10, не ре-анонсирует. Фильтр я оставил в конфигурации как страховку: если топология изменится и R15 начнёт их анонсировать, deny-строки сработают. Счётчики срабатываний в выводе prefix-list на этом IOL не показывает, о чём скажу в разделе 7.

## 5. Тестирование

Матрица пингов после всех изменений:

| Источник | Цель | Результат | Комментарий |
|---|---|---|---|
| R19 (10.0.0.19) | 10.0.0.14 | 5/5, ~1 ms | прямой линк 10.0.1.8/30 |
| R19 | 10.11.10.10 (VPC1) | 5/5, 2-7 ms | через O*IA 0.0.0.0/0 |
| R19 | 10.0.0.20 | 0/5 | у R14 нет маршрута: зона 0 разорвана |
| R20 (10.0.0.20) | 10.0.0.19 | 0/5 | маршрута нет: % Subnet not in table |
| R20 | 10.0.0.14 | 0/5 | маршрута нет, R15 не анонсирует loopback R14 в зону 102 |
| R20 | 10.0.0.15 | 5/5, ~1 ms | сосед по 10.0.1.20/30 |
| R20 | 10.11.10.10 (VPC1) | 5/5, 2-6 ms | через O IA 10.11.0.0/16 |
| VPC1 (10.11.10.10) | 10.0.0.12 | 5/5, ttl=255 | см. пояснение ttl ниже |
| VPC1 | 10.0.0.13 | 5/5, ttl=255 | см. пояснение ttl ниже |
| VPC1 | 10.0.0.14 | 5/5, ttl=254 | 2 перехода |
| VPC1 | 10.0.0.15 | 5/5, ttl=254 | 2 перехода |
| VPC1 | 10.0.0.19 | 5/5, ttl=253 | 3 перехода, вопреки ожиданию из плана |
| VPC1 | 10.0.0.20 | 5/5, ttl=253 | 3 перехода, вопреки ожиданию из плана |
| VPC7 (10.11.20.10) | 10.0.0.14 | 5/5, ttl=254 | |
| VPC7 | 10.0.0.15 | 5/5, ttl=254 | |

Пояснение по ttl у VPC1. В первом прогоне пингов ttl=255 у 10.0.0.13 выглядел как опечатка (R13 же не подключён к VLAN 10 напрямую?), поэтому я перепроверил все четыре адреса отдельно. Повторные пинги:

```text
VPCS> ping 10.0.0.12
84 bytes from 10.0.0.12 icmp_seq=1 ttl=255 time=1.737 ms
84 bytes from 10.0.0.12 icmp_seq=2 ttl=255 time=2.077 ms
84 bytes from 10.0.0.12 icmp_seq=3 ttl=255 time=2.091 ms
84 bytes from 10.0.0.12 icmp_seq=4 ttl=255 time=1.783 ms
84 bytes from 10.0.0.12 icmp_seq=5 ttl=255 time=2.043 ms

VPCS> ping 10.0.0.13
84 bytes from 10.0.0.13 icmp_seq=1 ttl=255 time=3.179 ms
84 bytes from 10.0.0.13 icmp_seq=2 ttl=255 time=3.160 ms
84 bytes from 10.0.0.13 icmp_seq=3 ttl=255 time=3.350 ms
84 bytes from 10.0.0.13 icmp_seq=4 ttl=255 time=3.103 ms
84 bytes from 10.0.0.13 icmp_seq=5 ttl=255 time=2.957 ms

VPCS> ping 10.0.0.19
84 bytes from 10.0.0.19 icmp_seq=1 ttl=253 time=3.620 ms
84 bytes from 10.0.0.19 icmp_seq=2 ttl=253 time=3.277 ms
84 bytes from 10.0.0.19 icmp_seq=3 ttl=253 time=3.106 ms
84 bytes from 10.0.0.19 icmp_seq=4 ttl=253 time=2.926 ms
84 bytes from 10.0.0.19 icmp_seq=5 ttl=253 time=3.213 ms

VPCS> ping 10.0.0.20
84 bytes from 10.0.0.20 icmp_seq=1 ttl=253 time=4.001 ms
84 bytes from 10.0.0.20 icmp_seq=2 ttl=253 time=2.945 ms
84 bytes from 10.0.0.20 icmp_seq=3 ttl=253 time=2.885 ms
84 bytes from 10.0.0.20 icmp_seq=4 ttl=253 time=3.384 ms
84 bytes from 10.0.0.20 icmp_seq=5 ttl=253 time=3.431 ms
```

Все четыре адреса прошли 5/5, значения ttl совпали. Ошибки не было, я просто не учёл, как выглядит адресация в работе 1: и R12, и R13 примыкают к L2-сегменту VLAN 10 через саб-интерфейсы e0/0.10 (10.11.10.2 и 10.11.10.3, там же HSRP, vIP 10.11.10.1 и есть шлюз VPC1). Loopback R12 (10.0.0.12/32) и loopback R13 (10.0.0.13/32) отвечают на пинг за один хоп от VPC1: ttl=255. Loopback'и R14 и R15 лежат за одним роутером глубже: ttl=254. Loopback'и R19 и R20 за двумя роутерами: ttl=253.

Два факта из матрицы, которые я закладывал в план как «проверить», и они получились не так, как я ожидал:

- R19 -> 10.0.0.20 не проходит (0/5): зона 0 разорвана, R14 loopback R20 не знает, маршрута нет. Это подтверждает разрыв backbone.
- R19 и R20 достижимы из зоны 10 (VPC1 пингует 10.0.0.19 и 10.0.0.20 с ttl=253): R14 как ABR зоны 101 анонсирует её сети в stub-зону 10, а stub-зона type 3 LSA принимает. Фильтр стоит только у R15 на входе в зону 102, на направление R14 зона 10 он не действует.

## 6. Что оказалось неочевидным

**Разорванная зона 0 меняет поведение ABR.** Пока R14 и R15 не в adjacency, каждый ABR «видит» зону 0 только своим loopback'ом. R14 анонсирует 10.0.0.14/32 в зону 10, R15 получает его как inter-area маршрут (на R15 он O IA [110/21], а не C). В обратную сторону R15 анонсирует 10.0.0.15/32, и R14 его получает. Loopback'и ABR друг другу «доходят» обходом через зону 10, а не через backbone.

**ABR не переносит type 3 LSA из одной зоны в другую «через голову».** R15 знает 10.0.0.19/32 и 10.0.1.8/30, потому что получил их через зону 10 (R14 анонсировал сети зоны 101 в зону 10). Но в зону 102 R15 их не публикует: для ABR они не «местные». Поэтому R20 не видит их и без фильтра, и пункт 4 задания выполняется механикой OSPF, а не моей командой. Фильтр остался страховкой.

**ABR не публикует в зону 102 loopback соседнего ABR (10.0.0.14/32)**, хотя сам его знает: префикс принадлежит зоне 0, adjacency в зоне 0 нет, и type 3 LSA из зоны 0 в базу зоны 102 не попадает. Отсюда 0/5 у R20 -> 10.0.0.14.

**Stub-зона принимает type 3 LSA.** В зоне 10 (stub) R12, R13 и я из VPC1 видим loopback'и R19 и R20: stub подавляет только external (type 5), а межзонные (type 3) проходят. Если бы зона 10 была totally stubby, VPC1 не пинговал бы .19 и .20.

**Команда range создаёт Null0-маршрут на ABR.** Суммарный 10.11.0.0/16 появился в таблице R15 как intra-area маршрут с next hop Null0 (metric 20). Это защита от «дыры» в суммаризации: пакет в 10.11.50.0/24 (существующий, но не объявленный) будет сброшен, а не уйдёт затылку.

**ttl в пингах считывается как глубина топологии.** 255, 254, 253 у целей из VPC1 ровно соответствуют числу переходов: 1 (R12/R13 в VLAN 10), 2 (R14/R15), 3 (R19/R20). Удобный способ проверить путь без трассировки, особенно когда у VPCS нет `trace` до нужной цели.

## 7. Ограничения

- Фильтр NO-AREA101 на стенде формальный: R15 в зону 102 сети зоны 101 не анонсировал и до фильтра, deny-строки не срабатывали. Оставил в конфиге как страховку, эффект продемонстрировать не на что.
- У этого образа IOL нет счётчиков matches в выводе `show ip prefix-list` и сама команда `show prefix-list NO-AREA101` не существует (Invalid input), работает только вариант с `ip`: `show ip prefix-list`.
- В running-config IOL команда `log-adjacency-changes` в секции `router ospf 1` не отображается (на этом образе), хотя вводилась. События adjacency я видел в логах консоли.
- IPv6 не настраивался: пункт 5 задания выполнен частично, вся логика зон на IPv4 (раздел 1).
- Virtual link не использовался: вне программы занятия, и через stub-зону невозможен (RFC 2328, раздел 3.6).
- `area 10 range` создал на ABR локальный маршрут с next hop Null0, в планировании я этого не закладывал: зафиксировал как есть (раздел 4.4).

## 8. Откат

На каждом из шести роутеров убрать процесс OSPF:

```text
no router ospf 1
```

На R12 и R13 дополнительно снять с интерфейсов остатки, если они остались после `no router ospf`:

```text
no ip ospf 1 area 10
```

На R15 удалить prefix-list:

```text
no ip prefix-list NO-AREA101
```

После отката `show ip route` на всех роутерах возвращается к connected-сетям из раздела 2. Конфигурацию HSRP и IP на саб-интерфейсах R12/R13 (работа 1) откат не трогает: там я добавлял только строки `ip ospf`.

## 9. Соответствие требованиям

| Требование | Реализация | Где |
|---|---|---|
| 1. R14 и R15 в зоне 0 | Loopback0 обоих ABR в area 0 | Раздел 4.2 |
| 2. R12 и R13 в зоне 10 плюс маршрут по умолчанию | `area 10 stub`, default вставляет ABR | Разделы 4.1, 5 |
| 3. R19 в зоне 101, только маршрут по умолчанию | `area 101 stub no-summary` на R14, `area 101 stub` на R19 | Разделы 4.1, 4.3 |
| 4. R20 в зоне 102, все маршруты, кроме сетей зоны 101 | `area 102 filter-list prefix NO-AREA101 in` на R15 плюс поведение ABR | Разделы 4.3, 4.5, 5 |
| 5. Логика повторяется для IPv6 | Не настраивалось, осознанное решение | Разделы 1, 7 |
| 6. План и изменения в документации | Этот README: план, конфиги, выводы проверок | Весь документ |

## Источники

- IETF, [RFC 2328](https://www.rfc-editor.org/rfc/rfc2328): OSPFv2, раздел 3.6 (virtual link), требования к структуре зон.
- Cisco, [OSPF ABR Type 3 LSA Filtering](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_ospf/configuration/15-e/iro-15-e-book/iro-abr-type-3.html): `area N filter-list prefix ... in/out`, prefix-list для межзонных LSA.
- Cisco, [OSPF Command Reference, area filter-list](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_ospf/command/iro-cr-book/m_ospf-a1.html): синтаксис и направления in/out.
- NetworkLessons, [Introduction to OSPF Stub Areas](https://networklessons.com/ospf/introduction-to-ospf-stub-areas): stub, totally stubby, NSSA, totally NSSA.
- NetworkLessons, [How to configure OSPF Summarization](https://networklessons.com/ospf/how-to-configure-ospf-summarization): `area N range`, суммаризация inter-area маршрутов.
