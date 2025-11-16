

## Что такое Underlay-сеть и роль OSPF в ней

**Underlay (Фундаментальная сеть)** — это базовая транспортная инфраструктура, которая обеспечивает IP-связность между всеми узлами будущей overlay-сети (VTEP - VXLAN Tunnel End Points).

**Роль OSPF в Underlay:** OSPF создает надежную, быструю и предсказуемую IP-сеть, поверх которой будут работать overlay-технологии (VXLAN, MPLS, DMVPN).

---

## Фундаментальные принципы OSPF для Underlay

### 1. Детерминированная маршрутизация
- **Цель:** Предсказуемые пути между любыми двумя точками
- **Метод:** Использование метрик (cost) на основе пропускной способности

### 2. Быстрая сходимость
- **Требование:** Восстановление за доли секунды при сбоях
- **Механизм:** Hello-таймеры, Dead-таймеры, SPF-расчет

### 3. Поддержка ECMP (Equal-Cost Multi-Path)
- **Важность:** Балансировка нагрузки и отказоустойчивость
- **Реализация:** Multiple paths с одинаковой метрикой

---

## Детальный процесс работы OSPF в Underlay

### Фаза 1: Установление соседства (Neighbor Adjacency)

#### Шаг 1.1: Обнаружение соседей
```bash
# Конфигурация на интерфейсе
interface Ethernet1/1
  ip address 10.1.1.1/30
  ip ospf network point-to-point
  ip ospf cost 10
  ip router ospf 1 area 0
```

**Процесс:**
1. **Hello-пакеты:** Рассылаются каждые 10 секунд (по умолчанию) на multicast-адрес **224.0.0.5**
2. **Параметры соседства:** Должны совпадать:
   - Area ID
   - Authentication (если используется)
   - Hello/Dead интервалы
   - MTU
   - Network Type

#### Шаг 1.2: Формирование смежности (Adjacency)
```bash
# Состояния OSPF:
# DOWN → INIT → 2-WAY → EXSTART → EXCHANGE → LOADING → FULL
```

**Критические состояния:**
- **2-WAY:** Базовое соседство установлено (видим друг друга в Hello)
- **FULL:** Полная синхронизация LSDB завершена

### Фаза 2: Синхронизация баз данных (Database Exchange)

#### Шаг 2.1: Обмен DBD пакетами
```
R1 → R2: DBD (Seq=100, I=1, M=1)  # Инициирование обмена
R2 → R1: DBD (Seq=200, I=1, M=1)  # Ответ с своим Seq
R1 → R2: DBD (Seq=200, I=0, M=0)  # Подтверждение
```

**Процесс Master/Slave:**
- Устройство с большим Router-ID становится Master
- Master управляет sequence numbers для синхронизации

#### Шаг 2.2: Запрос и передача LSA
```bash
# После DBD следует:
# LSR (Link State Request) → LSU (Link State Update) → LSAck
```

**Типы LSA критичные для Underlay:**

| Тип | Назначение | Область распространения |
|-----|------------|------------------------|
| **LSA Type 1** (Router LSA) | Описывает интерфейсы маршрутизатора | Только внутри Area |
| **LSA Type 2** (Network LSA) | Генерируется DR для multi-access сетей | Только внутри Area |

### Фаза 3: Расчет маршрутов (SPF Calculation)

#### Шаг 3.1: Построение графа сети
```python
# Псевдокод алгоритма Дейкстры
def dijkstra(graph, source):
    dist[source] = 0
    priority_queue = [(0, source)]
    
    while priority_queue:
        current_dist, current_node = heapq.heappop(priority_queue)
        
        for neighbor, weight in graph[current_node].items():
            distance = current_dist + weight
            if distance < dist[neighbor]:
                dist[neighbor] = distance
                heapq.heappush(priority_queue, (distance, neighbor))
```

#### Шаг 3.2: Заполнение таблицы маршрутизации
```bash
# Пример RIB после SPF
O       10.1.1.0/30 [110/10] via 10.2.2.2, Ethernet1/1
O       10.1.2.0/30 [110/20] via 10.2.2.2, Ethernet1/1
O       192.168.1.1/32 [110/30] via 10.2.2.2, Ethernet1/1
```

---

## Критичные настройки OSPF для Underlay

### 1. Point-to-Point линки
```bash
interface Ethernet1/1
  ip ospf network point-to-point  # Устраняет выбор DR/BDR
  ip ospf cost 10                 # Явное указание метрики
```

**Преимущества P2P:**
- Нет выбора DR/BDR
- Быстрее установление смежности
- Меньше LSA Type 2

### 2. Passive Interface
```bash
router ospf 1
  passive-interface default        # Все интерфейсы пассивны по умолчанию
  no passive-interface Ethernet1/1 # Явное включение OSPF на нужных
  no passive-interface Ethernet1/2
```

### 3. Loopback адресация
```bash
interface loopback0
  ip address 192.168.1.1/32        # /32 для однозначности
  ip ospf 1 area 0                 # Анонсирование loopback
```

---

## Практический пример построения Spine-Leaf Underlay

### Топология:
```
        Spine1 (1.1.1.1)
       /               \
Leaf1 (2.2.2.2)     Leaf2 (3.3.3.3)
```

### Конфигурация Spine1:
```bash
! Базовая конфигурация
hostname Spine1
ip routing

! Loopback интерфейс
interface loopback0
  ip address 1.1.1.1/32
  ip ospf 1 area 0

! Интерфейсы к Leaf
interface Ethernet1/1
  description to-Leaf1
  no switchport
  ip address 10.1.1.1/30
  ip ospf network point-to-point
  ip ospf cost 10
  ip router ospf 1 area 0

interface Ethernet1/2
  description to-Leaf2
  no switchport
  ip address 10.1.2.1/30
  ip ospf network point-to-point
  ip ospf cost 10
  ip router ospf 1 area 0

! OSPF процесс
router ospf 1
  router-id 1.1.1.1
  passive-interface default
  no passive-interface Ethernet1/1
  no passive-interface Ethernet1/2
  auto-cost reference-bandwidth 100000  # Для 100G+ линков
```

### Конфигурация Leaf1:
```bash
hostname Leaf1
ip routing

interface loopback0
  ip address 2.2.2.2/32
  ip ospf 1 area 0

interface Ethernet1/1
  description to-Spine1
  no switchport
  ip address 10.1.1.2/30
  ip ospf network point-to-point
  ip ospf cost 10
  ip router ospf 1 area 0

router ospf 1
  router-id 2.2.2.2
  passive-interface default
  no passive-interface Ethernet1/1
  auto-cost reference-bandwidth 100000
```

---

## Мониторинг и диагностика

### Ключевые команды диагностики:

```bash
# Проверка соседства
show ip ospf neighbor

# Детальная информация о соседе
show ip ospf neighbor detail

# База данных состояний каналов
show ip ospf database

# Расчетные маршруты
show ip ospf route

# Интерфейсы OSPF
show ip ospf interface

# Статистика SPF расчетов
show ip ospf statistics
```

### Пример вывода диагностики:
```bash
Spine1# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2          1   FULL/  -        00:00:35    10.1.1.2        Eth1/1
3.3.3.3          1   FULL/  -        00:00:33    10.1.2.2        Eth1/2

Spine1# show ip route ospf

O        2.2.2.2/32 [110/20] via 10.1.1.2, Ethernet1/1
O        3.3.3.3/32 [110/20] via 10.1.2.2, Ethernet1/2
```

---

## Решение распространенных проблем

### Проблема 1: Соседство не устанавливается
**Причины:**
- Несовпадение Area ID
- Разные Hello/Dead таймеры
- ACL блокируют multicast 224.0.0.5
- Неправильный network type

### Проблема 2: Маршруты не появляются
**Причины:**
- Passive interface на transit линках
- Фильтрация LSA
- Неправильные cost метрики

### Проблема 3: Медленная сходимость
**Решение:**
```bash
interface Ethernet1/1
  ip ospf dead-interval 1     # Быстрое обнаружение сбоев
  ip ospf hello-interval 250  # 250 ms hello
```

---

## Преимущества OSPF для Underlay

1. **Быстрая сходимость:** Sub-second convergence с оптимизированными таймерами
2. **ECMP поддержка:** Автоматическая балансировка нагрузки
3. **Предсказуемость:** Детерминированные пути на основе cost
4. **Масштабируемость:** Поддержка крупных сетей с иерархией areas
5. **Экосистема:** Широкая поддержка оборудованием и инструментами

## Ограничения

1. **Сложность:** Требует глубокого понимания для тонкой настройки
2. **Area иерархия:** Может создать излишнюю сложность в плоских сетях
3. **LSA flooding:** В очень крупных сетях может создавать нагрузку

OSPF остается надежным и проверенным выбором для построения Underlay-сетей, особенно в environments где команда имеет опыт работы с этим протоколом.

Подробно разберем построение Underlay-сети с использованием **IS-IS** — мощного протокола состояния каналов, который особенно популярен в крупных дата-центрах и сетях провайдеров.

## Что такое IS-IS и его роль в Underlay

**IS-IS (Intermediate System to Intermediate System)** — это протокол состояния каналов, который изначально был разработан для сетей OSI, но был адаптирован для IP-сетей.

**Особенности IS-IS для Underlay:**
- Работает на уровне 2 (Data Link Layer)
- Не зависит от IP, что обеспечивает большую гибкость
- Идеален для крупных, плоских сетей дата-центров

---

## Фундаментальные принципы IS-IS для Underlay

### 1. Архитектура уровней (Levels)
```
Level 2 (L2) Backbone
    ↑
Level 1 (L1) Areas
```

- **Level 1 (L1):** Маршрутизация внутри area
- **Level 2 (L2):** Маршрутизация между areas
- **Level 1-2 (L1-L2):** Устройства, работающие на обоих уровнях

### 2. NET Address (Network Entity Title)
**Формат:** `AA.AAAA.BBBB.BBBB.BBBB.00`
- **AA:** AFI (обычно 49 для private)
- **AAAA:** Area ID
- **BBBB.BBBB.BBBB:** System ID (6 bytes)
- **00:** SEL (всегда 00 для сетевых устройств)

---

## Детальный процесс работы IS-IS в Underlay

### Фаза 1: Установление соседства (Adjacency)

#### Шаг 1.1: Обмен Hello-пакетами
```bash
# Конфигурация базового IS-IS
router isis UNDERLAY
 net 49.0001.0000.0000.0001.00
 is-type level-2-only
 metric-style wide
```

**Типы Hello-пакетов:**
- **IIH (IS-IS Hello):** Для установления соседства
- **L1 IIH:** Для Level 1 соседей
- **L2 IIH:** Для Level 2 соседей

#### Шаг 1.2: Three-way Handshake
```
R1 → R2: IIH (State=Init)
R2 → R1: IIH (State=Up, видит R1)
R1 → R2: IIH (State=Up, видит R2) → Adjacency Established
```

**Состояния смежности:**
- **Down:** Соседство не установлено
- **Initial:** Получен Hello, но без нашего ID
- **Up:** Полное соседство установлено

### Фаза 2: Распространение LSP (Link State PDU)

#### Шаг 2.1: Генерация LSP
```bash
# Каждое устройство генерирует свои LSP:
# - LSP ID: System ID + Pseudonode ID (00 для обычных LSP)
# - Sequence Number: Для определения актуальности
# - Lifetime: Время жизни LSP
```

**Типы LSP для Underlay:**
- **Non-pseudonode LSP:** Описывает маршрутизатор и его подключения
- **Pseudonode LSP:** Для broadcast сетей (генерируется DIS)

#### Шаг 2.2: Flooding механизм
```python
# Процесс распространения LSP
def lsp_flooding(received_lsp):
    if received_lsp.seq > local_lsp.seq:
        update_local_database(received_lsp)
        forward_to_all_neighbors_except_sender(received_lsp)
    elif received_lsp.seq == local_lsp.seq:
        # LSP актуален - ничего не делать
        pass
    else:
        # Получен устаревший LSP - отправить актуальный обратно
        send_local_lsp_to_sender()
```

### Фаза 3: Расчет маршрутов (SPF Calculation)

#### Шаг 3.1: Построение графа на основе TLV
**TLV (Type-Length-Value) поля в LSP:**
- **TLV 22:** Extended IS Reachability (метрики)
- **TLV 135:** Extended IP Reachability (IP префиксы)
- **TLV 229:** Multi-Topology (для различных классов обслуживания)

#### Шаг 3.2: Алгоритм Дейкстры
```python
def isis_spf_calculation():
    # Инициализация
    tent = []  # Tentative list
    paths = {} # Shortest paths
    
    # Добавление исходного узла
    tent.append((local_system, 0, []))
    
    while tent:
        # Выбор узла с минимальной метрикой
        current, cost, path = min(tent, key=lambda x: x[1])
        tent.remove((current, cost, path))
        
        # Добавление в окончательные пути
        paths[current] = (cost, path + [current])
        
        # Обработка соседей
        for neighbor, link_cost in get_neighbors(current):
            if neighbor not in paths:
                tent.append((neighbor, cost + link_cost, path + [current]))
```

---

## Критичные настройки IS-IS для Underlay

### 1. Базовая конфигурация
```bash
! Включение IS-IS процесса
router isis UNDERLAY
 net 49.0001.0000.0000.0001.00
 is-type level-2-only
 metric-style wide
 log-adjacency-changes
```

### 2. Настройка интерфейсов
```bash
interface Ethernet1/1
 description to-Spine1
 no switchport
 ip address 10.1.1.1/30
 ip router isis UNDERLAY
 isis circuit-type level-2-only
 isis metric 10
 isis network point-to-point
```

### 3. Оптимизация для Underlay
```bash
router isis UNDERLAY
 ! Быстрые таймеры для быстрой сходимости
 hello-interval 333 ms
 hello-multiplier 3
 ! Оптимизация SPF
 spf-interval 50 50 500  # initial, second, maximum wait
 lsp-gen-interval 50 50 500
```

---

## Практический пример построения Spine-Leaf Underlay с IS-IS

### Топология:
```
        Spine1 (0000.0000.0001)
       /               \
Leaf1 (0000.0000.0002)  Leaf2 (0000.0000.0003)
```

### Конфигурация Spine1:
```bash
hostname Spine1

! Базовая конфигурация IS-IS
router isis UNDERLAY
 net 49.0001.0000.0000.0001.00
 is-type level-2-only
 metric-style wide
 log-adjacency-changes
 nsf cisco
 address-family ipv4 unicast
  metric-style wide
  maximum-paths 4
 exit-address-family

! Loopback интерфейс
interface loopback0
 ip address 192.168.1.1/32
 ip router isis UNDERLAY
 isis circuit-type level-2-only

! Интерфейсы к Leaf коммутаторам
interface Ethernet1/1
 description to-Leaf1
 no switchport
 ip address 10.1.1.1/30
 ip router isis UNDERLAY
 isis circuit-type level-2-only
 isis metric 10
 isis network point-to-point
 isis hello-interval 333
 isis hello-multiplier 3

interface Ethernet1/2
 description to-Leaf2
 no switchport
 ip address 10.1.2.1/30
 ip router isis UNDERLAY
 isis circuit-type level-2-only
 isis metric 10
 isis network point-to-point
 isis hello-interval 333
 isis hello-multiplier 3
```

### Конфигурация Leaf1:
```bash
hostname Leaf1

router isis UNDERLAY
 net 49.0001.0000.0000.0002.00
 is-type level-2-only
 metric-style wide
 log-adjacency-changes
 address-family ipv4 unicast
  metric-style wide
  maximum-paths 4
 exit-address-family

interface loopback0
 ip address 192.168.1.2/32
 ip router isis UNDERLAY
 isis circuit-type level-2-only

interface Ethernet1/1
 description to-Spine1
 no switchport
 ip address 10.1.1.2/30
 ip router isis UNDERLAY
 isis circuit-type level-2-only
 isis metric 10
 isis network point-to-point
 isis hello-interval 333
 isis hello-multiplier 3
```

---

## DIS (Designated IS) в Broadcast сетях

### Роль DIS:
- Генерация Pseudonode LSP для broadcast сегментов
- Координация flooding в multi-access сетях

### Выбор DIS:
```bash
# Приоритет DIS (по умолчанию 64)
interface Ethernet1/1
 isis priority 100  # Более высокий приоритет становится DIS
```

**Критерии выбора DIS:**
1. Наибольший приоритет (0-127)
2. Наибольший MAC-адрес (если приоритеты равны)

---

## Мониторинг и диагностика

### Ключевые команды диагностики:

```bash
# Проверка соседства
show isis neighbors
show isis neighbors detail

# База данных LSP
show isis database
show isis database detail

# Статистика интерфейсов
show isis interface
show isis interface detail

# Таблица маршрутизации IS-IS
show isis route
show isis topology

# SPF статистика
show isis spf-log
show isis spf-tree

# LSP статистика
show isis lsp-log
```

### Пример вывода диагностики:
```bash
Spine1# show isis neighbors

System Id       Type Interface   IP Address      State Holdtime Circuit Id
0000.0000.0002  L2   Et1/1       10.1.1.2        UP    23       01
0000.0000.0003  L2   Et1/2       10.1.2.2        UP    25       01

Spine1# show isis database

IS-IS Level-2 Link State Database:
LSPID                 LSP Seq Num  LSP Checksum  LSP Holdtime  ATT/P/OL
0000.0000.0001.00-00  0x0000000C   0x8A2F        1023          0/0/0
0000.0000.0002.00-00  0x00000009   0xF3A1        1187          0/0/0
0000.0000.0003.00-00  0x00000008   0xB2C4        1123          0/0/0
```

---

## Преимущества IS-IS для Underlay

### 1. **Масштабируемость**
- Нет ограничений Area 0 как в OSPF
- Более эффективное flooding LSP
- Поддержка очень крупных плоских сетей

### 2. **Гибкость**
- Независимость от IP (работает на L2)
- Легкое добавление новых areas
- Поддержка Multi-Topology Routing

### 3. **Производительность**
- Быстрая сходимость с оптимизированными таймерами
- Эффективный механизм flooding
- Минимальные overhead

### 4. **Отказоустойчивость**
- NSF (Non-Stop Forwarding) поддержка
- Быстрое перераспределение трафика при сбоях
- Стабильные ECMP пути

---

## Сравнение с OSPF для Underlay

| Параметр | IS-IS | OSPF |
|----------|-------|------|
| **Архитектура** | Уровни (Levels) | Areas |
| **Transport** | L2 (независим от IP) | IP (протокол 89) |
| **База данных** | Единая для всех уровней | Раздельная по areas |
| **Расширяемость** | Легкое добавление уровней | Сложное расширение backbone |
| **DIS/DR** | DIS может меняться | DR выбор более стабилен |
| **Конфигурация** | Более сложная начальная настройка | Более интуитивная |

---

## Best Practices для IS-IS Underlay

### 1. **Адресация System ID**
```bash
# Использование MAC-адресов или запланированной схемы
Spine1: 0000.0000.0001
Spine2: 0000.0000.0002
Leaf1:  0000.0000.0011
Leaf2:  0000.0000.0012
```

### 2. **Метрики**
```bash
# Явное указание метрик для предсказуемости
interface Ethernet1/1
 isis metric 10  # 10G линк

interface Ethernet1/2
 isis metric 40  # 1G линк
```

### 3. **Безопасность**
```bash
# Аутентификация
interface Ethernet1/1
 isis password securepassword level-2
```

### 4. **Оптимизация производительности**
```bash
router isis UNDERLAY
 # Быстрая сходимость
 spf-interval 50 50 500
 lsp-gen-interval 50 50 500
 # ECMP
 maximum-paths 8
```

IS-IS является отличным выбором для построения Underlay-сетей в крупных дата-центрах, обеспечивая превосходную масштабируемость, производительность и гибкость по сравнению с традиционными протоколами.