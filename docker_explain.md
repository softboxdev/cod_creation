# Как устроены Docker контейнеры: архитектура и процессы

## 1. Архитектурная диаграмма Docker

```mermaid
graph TB
    subgraph "Хост-система (Host OS)"
        subgraph "Docker Engine"
            DockerDaemon[Docker Daemon<br/>dockerd]
            REST_API[REST API]
            Containerd[ContainerD]
            Runc[RunC]
        end
        
        subgraph "Сетевые драйверы"
            Bridge[Bridge Network]
            Overlay[Overlay Network]
            Host[Host Network]
        end
        
        subgraph "Файловая система"
            UnionFS[Union File Systems<br/>overlay2, aufs, btrfs]
            Volumes[Docker Volumes]
            BindMounts[Bind Mounts]
        end
    end
    
    subgraph "Контейнер 1"
        App1[Приложение 1<br/>+ зависимости]
        Libs1[Библиотеки]
        FS1[Файловая система<br/>контейнера 1]
        NS1[Namespace<br/>PID, NET, IPC, MNT, UTS]
        Cgroups1[Cgroups<br/>Ограничения CPU, RAM]
    end
    
    subgraph "Контейнер 2"
        App2[Приложение 2<br/>+ зависимости]
        Libs2[Библиотеки]
        FS2[Файловая система<br/>контейнера 2]
        NS2[Namespace<br/>PID, NET, IPC, MNT, UTS]
        Cgroups2[Cgroups<br/>Ограничения CPU, RAM]
    end
    
    DockerCli[Docker Client<br/>docker CLI] --> REST_API
    REST_API --> DockerDaemon
    DockerDaemon --> Containerd
    Containerd --> Runc
    Runc --> Контейнер1
    Runc --> Контейнер2
    
    Контейнер1 --> UnionFS
    Контейнер2 --> UnionFS
    Контейнер1 --> Bridge
    Контейнер2 --> Bridge
    Контейнер1 --> Volumes
    Контейнер2 --> Volumes
```

## 2. Диаграмма последовательности: Жизненный цикл контейнера

```mermaid
sequenceDiagram
    participant User as Пользователь
    participant CLI as Docker CLI
    participant Daemon as Docker Daemon
    participant Containerd
    participant Runc
    participant Kernel as Ядро Linux
    participant Registry as Docker Registry
    
    User->>CLI: docker run nginx:latest
    CLI->>Daemon: POST /containers/create
    Daemon->>Registry: Проверка локального образа
    
    alt Образ отсутствует локально
        Registry-->>Daemon: Образ не найден
        Daemon->>Registry: pull nginx:latest
        Registry-->>Daemon: Слои образа
        Daemon->>Daemon: Сборка Union FS
    else Образ присутствует
        Registry-->>Daemon: Образ найден
    end
    
    Daemon->>Containerd: create container
    Containerd->>Runc: create
    Runc->>Kernel: 1. Создание Namespaces
    Note over Runc,Kernel: PID, NET, IPC,<br/>UTS, MNT, USER
    Runc->>Kernel: 2. Создание Cgroups
    Note over Runc,Kernel: Ограничения CPU,<br/>памяти, I/O
    Runc->>Kernel: 3. Настройка сети
    Note over Runc,Kernel: Создание veth pair,<br/>добавление в bridge
    Runc->>Kernel: 4. Монтирование FS
    Note over Runc,Kernel: UnionFS + Copy-on-Write
    Runc->>Containerd: container created
    Containerd->>Daemon: container created
    
    User->>CLI: docker start nginx
    CLI->>Daemon: POST /containers/start
    Daemon->>Containerd: start container
    Containerd->>Runc: run
    Runc->>Kernel: Запуск init процесса
    Kernel-->>Runc: PID 1 запущен
    Runc-->>Containerd: container running
    Containerd-->>Daemon: container running
    Daemon-->>CLI: Container started
    CLI-->>User: Контейнер запущен
    
    User->>CLI: docker exec -it nginx bash
    CLI->>Daemon: POST /containers/exec
    Daemon->>Containerd: exec process
    Containerd->>Runc: exec
    Runc->>Kernel: Создание процесса<br/>в существующих namespaces
    Kernel-->>Runc: Процесс создан
    Runc-->>Containerd: exec success
    Containerd-->>Daemon: exec success
    Daemon-->>CLI: TTY подключен
```

## 3. Диаграмма слоев файловой системы Docker

```mermaid
graph TD
    subgraph "Слоистая файловая система (UnionFS)"
        subgraph "Контейнерный слой (R/W)"
            C1[Контейнер 1<br/>Read/Write Layer]
            C2[Контейнер 2<br/>Read/Write Layer]
        end
        
        subgraph "Слои образа (Read Only)"
            L4[Слой 4: apt install python<br/>ID: a1b2c3]
            L3[Слой 3: apt update<br/>ID: x9y8z7]
            L2[Слой 2: COPY app.py /app<br/>ID: p5q6r7]
            L1[Базовый образ Ubuntu<br/>ID: d4e5f6]
        end
    end
    
    subgraph "Хранилище томов"
        V1[Том 1<br/>/var/lib/mysql]
        V2[Том 2<br/>/app/data]
    end
    
    subgraph "Bind Mounts"
        BM1[Хост: /home/user/data<br/>→ Контейнер: /data]
    end
    
    C1 --> L4
    C2 --> L4
    L4 --> L3
    L3 --> L2
    L2 --> L1
    
    C1 -.-> V1
    C2 -.-> V2
    C1 -.-> BM1
```

## 4. Диаграмма сетевой архитектуры Docker

```mermaid
graph TB
    subgraph "Хост-система"
        Eth0[eth0: 192.168.1.100]
        Docker0[docker0 bridge<br/>172.17.0.1/16]
        
        subgraph "Network Namespace 1"
            Veth1[veth12345@if2]
            EthC1[eth0@контейнер<br/>172.17.0.2/16]
            Container1[Контейнер 1<br/>nginx:80]
        end
        
        subgraph "Network Namespace 2"
            Veth2[veth67890@if2]
            EthC2[eth0@контейнер<br/>172.17.0.3/16]
            Container2[Контейнер 2<br/>app:5000]
        end
    end
    
    subgraph "Внешняя сеть"
        Internet[Интернет]
        Client[Клиент<br/>192.168.1.50]
    end
    
    Client --> Eth0
    Eth0 --> Docker0
    Docker0 --> Veth1
    Docker0 --> Veth2
    Veth1 --> EthC1
    Veth2 --> EthC2
    
    EthC1 --> Container1
    EthC2 --> Container2
    
    Container1 -- "Внутренняя связь" --> Container2
    
    %% iptables правила
    subgraph "IPTables правила"
        NAT[SNAT/MASQUERADE<br/>для исходящего трафика]
        DNAT[DNAT<br/>-p tcp --dport 8080<br/>→ 172.17.0.2:80]
        FORWARD[FORWARD цепочка<br/>между bridge и eth0]
    end
    
    Docker0 -.-> NAT
    Docker0 -.-> DNAT
    Docker0 -.-> FORWARD
```

## 5. Диаграмма последовательности: Построение образа Docker

```mermaid
sequenceDiagram
    participant Dev as Разработчик
    participant Build as Docker Build
    participant Builder as BuildKit
    participant Cache as Кэш слоев
    participant Registry
    
    Dev->>Build: docker build -t myapp:v1 .
    
    Build->>Builder: Чтение Dockerfile
    Builder->>Builder: Анализ инструкций
    
    loop Для каждой инструкции в Dockerfile
        Builder->>Cache: Проверка кэша слоя
        alt Слой есть в кэше
            Cache-->>Builder: Использовать кэш
            Note over Builder: Пропуск шага
        else Слоя нет в кэше
            Builder->>Builder: Выполнение инструкции
            Builder->>Cache: Сохранение слоя
            Note over Builder: Создан новый слой<br/>с уникальным ID
        end
    end
    
    Builder->>Builder: Создание manifest
    Builder->>Builder: Формирование образа
    
    Builder-->>Build: Образ собран
    Build-->>Dev: Successfully built myapp:v1
    
    Dev->>Build: docker push myapp:v1
    Build->>Registry: Аутентификация
    Build->>Registry: Загрузка слоев
    loop Загрузка каждого слоя
        Build->>Registry: PUSH слой (digest)
        Registry->>Registry: Проверка существования
        alt Слой уже существует
            Registry-->>Build: Слой пропущен
        else Новый слой
            Registry-->>Build: Слой принят
        end
    end
    Build->>Registry: Загрузка manifest
    Registry-->>Build: Образ опубликован
```

## Объяснение ключевых компонентов:

### 1. **Namespaces** (пространства имен)
- **PID namespace**: Изоляция процессов (контейнер видит только свои процессы)
- **NET namespace**: Изоляция сети (свои интерфейсы, IP, таблицы маршрутизации)
- **IPC namespace**: Изоляция межпроцессного взаимодействия
- **MNT namespace**: Изоляция точек монтирования файловой системы
- **UTS namespace**: Изоляция hostname и domainname
- **USER namespace**: Изоляция UID/GID (отображение пользователей)

### 2. **Cgroups** (control groups)
- Ограничение ресурсов:
  - CPU shares и quotas
  - Memory limits и swap
  - I/O bandwidth
  - Network priority
- Учет потребления ресурсов
- Заморозка/возобновление процессов

### 3. **Union File System**
- Слоистая архитектура:
  - Базовый слой (образ ОС)
  - Слои зависимостей (apt install, npm install)
  - Слой приложения (COPY, ADD)
  - Контейнерный слой (read-write)
- Copy-on-Write (CoW):
  - Слои только для чтения
  - Изменения записываются в верхний R/W слой

### 4. **Сетевая модель**
- Bridge сеть (docker0):
  - NAT для исходящих соединений
  - Порт форвардинг для входящих
- Host сеть:
  - Использование сетевого стека хоста
- Overlay сети:
  - Для Docker Swarm кластеров
- Macvlan:
  - Прямое выделение MAC адресов

### 5. **Архитектурные компоненты**
- **Docker CLI**: Интерфейс командной строки
- **Docker Daemon** (dockerd): Фоновый процесс, управляющий контейнерами
- **containerd**: Менеджер жизненного цикла контейнеров
- **runc**: Спецификация OCI, низкоуровневый рантайм
- **BuildKit**: Система сборки образов

### 6. **Изоляция vs Виртуализация**
```
Виртуализация (VM):           Контейнеры (Docker):
+------------------+          +------------------+
|     App A        |          |     App A        |
+------------------+          +------------------+
|   Библиотеки     |          |   Библиотеки     |
+------------------+          +------------------+
|   Guest OS A     |          |   Container      |
+------------------+          |     Runtime      |
|   Hypervisor     |          +------------------+
+------------------+          |    Host OS       |
|   Host OS        |          +------------------+
+------------------+          |   Hardware       |
|   Hardware       |          +------------------+
+------------------+
```

**Ключевые отличия:**
- Контейнеры разделяют ядро хоста
- Запуск за секунды (не минуты)
- Минимальный оверхед (1-5% vs 10-20% у VM)
- Портабельность: "build once, run anywhere"

Эта архитектура делает Docker легковесным, быстрым и портативным решением для контейнеризации приложений.