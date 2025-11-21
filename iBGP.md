
## 1. Что такое iBGP?

**iBGP (Internal BGP)** - это протокол для обмена маршрутной информацией **внутри одной автономной системы (AS)**.

## 2. Основная архитектура iBGP

```mermaid
flowchart TD
    A[Автономная система AS 65001] --> B[Топология полной ячеистой сети<br>Full Mesh]
    
    B --> C[Маршрутизатор A]
    B --> D[Маршрутизатор B]
    B --> E[Маршрутизатор C]
    B --> F[Маршрутизатор D]
    
    C --> G[iBGP сессии<br>со всеми маршрутизаторами]
    D --> G
    E --> G
    F --> G
    
    H[Внешняя сеть] --> I[eBGP сессии<br>только через<br>граничные маршрутизаторы]
    I --> C
    I --> F
```

## 3. Процесс установления iBGP сессии

```mermaid
sequenceDiagram
    participant A as Маршрутизатор A
    participant B as Маршрутизатор B
    
    Note over A,B: Этап 1: Конфигурация
    A->>A: router bgp 65001<br>neighbor B_IP remote-as 65001
    B->>B: router bgp 65001<br>neighbor A_IP remote-as 65001
    
    Note over A,B: Этап 2: Установление TCP сессии
    A->>B: TCP SYN (порт 179)
    B->>A: TCP SYN-ACK
    A->>B: TCP ACK
    Note over A,B: TCP соединение установлено
    
    Note over A,B: Этап 3: Обмен BGP сообщениями
    A->>B: OPEN message
    B->>A: OPEN message
    Note over A,B: Параметры согласованы
    
    A->>B: KEEPALIVE
    B->>A: KEEPALIVE
    Note over A,B: iBGP сессия установлена
    
    A->>B: UPDATE (маршруты)
    B->>A: UPDATE (маршруты)
```

## 4. Правило синхронизации iBGP

```mermaid
flowchart TD
    A[Маршрутизатор получает<br>внешний маршрут через eBGP] --> B{Правило синхронизации}
    
    B --> C[Включено sync]
    C --> D{Маршрут есть в IGP?}
    D -->|Да| E[Разрешить<br>адвертайз в iBGP]
    D -->|Нет| F[Запретить<br>адвертайз в iBGP]
    
    B --> G[Выключено sync<br>современные сети]
    G --> H[Всегда разрешить<br>адвертайз в iBGP]
```

## 5. Процесс распространения маршрута через iBGP

```mermaid
flowchart TD
    A[Граничный маршрутизатор] --> B[Получает маршрут от eBGP соседа]
    B --> C[Добавляет свой AS_PATH<br>и другие атрибуты]
    C --> D[Обновляет локальную RIB]
    
    D --> E[Распространение в iBGP]
    E --> F{Правило iBGP: <br>Не адвертайзить<br>iBGP-learned routes<br>другим iBGP соседям}
    
    F --> G[Решение: Full Mesh<br>или Route Reflector]
    
    G --> H[Вариант 1: Full Mesh]
    H --> I[Адвертайз напрямую<br>всем iBGP соседям]
    
    G --> J[Вариант 2: Route Reflector]
    J --> K[Адвертайз только<br>Route Reflector'у]
    K --> L[Route Reflector<br>реплицирует всем клиентам]
```

## 6. Сравнение iBGP и eBGP

```mermaid
flowchart LR
    subgraph AS1 [Автономная система 65001]
        A[Маршрутизатор A]
        B[Маршрутизатор B]
        C[Маршрутизатор C]
    end
    
    subgraph AS2 [Автономная система 65002]
        D[Маршрутизатор D]
        E[Маршрутизатор E]
    end
    
    A <-->|iBGP<br>Same AS| B
    B <-->|iBGP<br>Same AS| C
    A <-->|iBGP<br>Same AS| C
    
    A <-->|eBGP<br>Different AS| D
    C <-->|eBGP<br>Different AS| E
```

## 7. Атрибуты BGP в iBGP

```mermaid
flowchart TD
    A[BGP UPDATE Message] --> B[Атрибуты пути]
    
    B --> C[AS_PATH]
    C --> D[iBGP: Не добавляет AS<br>eBGP: Добавляет AS]
    
    B --> E[NEXT_HOP]
    E --> F[iBGP: Сохраняет оригинал<br>eBGP: Меняет на свой IP]
    
    B --> G[LOCAL_PREF]
    G --> H[Только в iBGP<br>Высокий = предпочтительный]
    
    B --> I[MED]
    I --> J[Адвертайзится в iBGP<br>Сравнивается между разными AS]
```

## 8. Процесс принятия решений в iBGP

```mermaid
flowchart TD
    A[Получен BGP маршрут] --> B[Валидация маршрута]
    B --> C{Валидный?}
    C -->|Нет| D[Отклонить]
    
    C -->|Да| E[BGP Decision Process]
    
    E --> F[1. Highest Weight<br>Cisco specific]
    E --> G[2. Highest LOCAL_PREF]
    E --> H[3. Local originated]
    E --> I[4. Shortest AS_PATH]
    E --> J[5. Lowest Origin type]
    E --> K[6. Lowest MED]
    E --> L[7. eBGP > iBGP]
    E --> M[8. Lowest IGP metric]
    E --> N[9. Oldest path]
    E --> O[10. Router ID]
    E --> P[11. Neighbor IP]
    
    Q[Выбран лучший путь] --> R[Добавить в RIB]
    R --> S[Адвертайзовать соседям]
```

## 9. Пример работы Route Reflector

```mermaid
flowchart TD
    A[Route Reflector] --> B[Клиент 1]
    A --> C[Клиент 2]
    A --> D[Клиент 3]
    
    E[Неклиентский сосед] --> A
    A --> E
    
    B --> F[Получает маршрут от eBGP]
    F --> B[Адвертайзит Route Reflector]
    B --> A
    
    A --> G[Рефлектирует всем клиентам]
    A --> C
    A --> D
    
    A --> H[Рефлектирует неклиентам<br>по стандартным правилам]
    A --> E
```

## 10. Полный процесс обработки iBGP маршрута

```mermaid
flowchart TD
    A[Начало] --> B[Получен BGP UPDATE]
    B --> C[Парсинг атрибутов]
    
    C --> D{Тип соседа?}
    D -->|eBGP| E[Обработка eBGP]
    D -->|iBGP| F[Обработка iBGP]
    
    F --> G[Проверка правила<br>синхронизации]
    G --> H{Синхронизация<br>требуется?}
    
    H -->|Да| I{Маршрут в IGP?}
    I -->|Нет| J[Отклонить маршрут]
    I -->|Да| K[Принять маршрут]
    
    H -->|Нет| K
    
    K --> L[BGP Decision Process]
    L --> M[Выбор лучшего пути]
    M --> N[Обновление RIB]
    N --> O[Адвертайз соседям]
    
    O --> P{iBGP learned?}
    P -->|Да| Q[Адвертайз только<br>eBGP соседям]
    P -->|Нет| R[Адвертайз всем<br>соседям]
```

## 11. Практический пример конфигурации

```bash
! Маршрутизатор A в AS 65001
router bgp 65001
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 ! iBGP соседи
 neighbor 192.168.1.2 remote-as 65001
 neighbor 192.168.1.3 remote-as 65001
 ! eBGP сосед
 neighbor 10.1.1.2 remote-as 65002
 ! Отключение синхронизации
 no synchronization
 ! Политика сети
 network 192.168.1.0 mask 255.255.255.0
```

## Ключевые особенности iBGP:

1. **Тот же AS номер** у всех участников
2. **Full Mesh требование** (без Route Reflector)
3. **Не изменяет AS_PATH** для внутренних маршрутов
4. **Сохраняет NEXT_HOP** от eBGP соседа
5. **Использует LOCAL_PREF** для управления трафиком внутри AS

iBGP обеспечивает согласованное распространение внешней маршрутной информации внутри автономной системы, работая поверх IGP (OSPF, EIGRP), который обеспечивает внутреннюю связность.