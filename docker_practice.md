# Практическая работа по освоению Docker в Ubuntu 24.04

## Цель работы
Ознакомиться с основами работы с Docker: установка, основные команды, создание и управление контейнерами, работа с образами и Dockerfile.

---

## Подготовительный этап

### 1. Установка Docker

```bash
# 1. Обновление пакетов
sudo apt update
sudo apt upgrade -y

# 2. Установка необходимых зависимостей
sudo apt install -y ca-certificates curl gnupg lsb-release

# 3. Добавление официального GPG-ключа Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 4. Добавление репозитория Docker
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 5. Установка Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 6. Проверка установки
sudo docker --version
```

### 2. Настройка прав пользователя (опционально)

```bash
# Добавление текущего пользователя в группу docker
sudo usermod -aG docker $USER

# Выйти и зайти обратно в систему для применения изменений
# или выполнить:
newgrp docker
```

---

## Практические задания

### Задание 1: Работа с базовыми командами Docker

```bash
# 1. Проверка состояния Docker
sudo systemctl status docker

# 2. Запуск простого контейнера
sudo docker run hello-world

# 3. Просмотр всех запущенных контейнеров
sudo docker ps

# 4. Просмотр всех контейнеров (включая остановленные)
sudo docker ps -a

# 5. Просмотр скачанных образов
sudo docker images

# 6. Получение информации о системе Docker
sudo docker info
```

**Вопросы для самопроверки:**
1. Что произошло после выполнения команды `docker run hello-world`?
2. Какой статус у контейнера hello-world сейчас?
3. Сколько места занимает образ hello-world?

### Задание 2: Работа с контейнерами

```bash
# 1. Запуск контейнера с Nginx
sudo docker run -d --name my-nginx -p 8080:80 nginx

# 2. Проверка работы Nginx
curl http://localhost:8080

# 3. Просмотр логов контейнера
sudo docker logs my-nginx

# 4. Остановка контейнера
sudo docker stop my-nginx

# 5. Запуск остановленного контейнера
sudo docker start my-nginx

# 6. Перезагрузка контейнера
sudo docker restart my-nginx

# 7. Подключение к терминалу запущенного контейнера
sudo docker exec -it my-nginx bash
# Внутри контейнера:
#   ls -la
#   exit

# 8. Удаление контейнера (сначала остановить)
sudo docker stop my-nginx
sudo docker rm my-nginx
```

**Вопросы для самопроверки:**
1. Что означает параметр `-p 8080:80`?
2. Какой командой можно увидеть процессы внутри контейнера?
3. Что происходит при удалении контейнера с данными?

### Задание 3: Работа с образами

```bash
# 1. Поиск образов в Docker Hub
sudo docker search ubuntu

# 2. Загрузка образа Ubuntu
sudo docker pull ubuntu:22.04

# 3. Запуск контейнера с интерактивным терминалом
sudo docker run -it --name my-ubuntu ubuntu:22.04 bash
# Внутри контейнера:
#   apt update
#   apt install -y python3
#   python3 --version
#   exit

# 4. Сохранение изменений в новый образ
sudo docker commit my-ubuntu my-ubuntu-python

# 5. Пометка образа для загрузки в репозиторий
sudo docker tag my-ubuntu-python my-docker-username/my-ubuntu-python:v1.0

# 6. Удаление образа
sudo docker rmi ubuntu:22.04
```

**Вопросы для самопроверки:**
1. Чем отличается `docker pull` от `docker run`?
2. Что сохраняется при коммите контейнера?
3. Как восстановить удаленный образ?

### Задание 4: Создание Dockerfile

```bash
# 1. Создание рабочей директории
mkdir ~/docker-project && cd ~/docker-project

# 2. Создание Dockerfile
cat > Dockerfile << 'EOF'
# Используем базовый образ
FROM ubuntu:22.04

# Обновление пакетов и установка Python
RUN apt update && \
    apt install -y python3 python3-pip && \
    apt clean && \
    rm -rf /var/lib/apt/lists/*

# Создание рабочей директории
WORKDIR /app

# Копирование файлов приложения
COPY . .

# Установка зависимостей Python
RUN pip3 install flask

# Открытие порта
EXPOSE 5000

# Команда для запуска
CMD ["python3", "app.py"]
EOF

# 3. Создание простого приложения на Flask
cat > app.py << 'EOF'
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello from Docker Container!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# 4. Сборка образа
sudo docker build -t my-flask-app .

# 5. Запуск контейнера
sudo docker run -d -p 5000:5000 --name flask-container my-flask-app

# 6. Проверка работы
curl http://localhost:5000
```

### Задание 5: Работа с томами (Volumes)

```bash
# 1. Создание тома
sudo docker volume create my-data

# 2. Просмотр списка томов
sudo docker volume ls

# 3. Запуск контейнера с примонтированным томом
sudo docker run -it --name volume-test -v my-data:/data ubuntu:22.04 bash
# Внутри контейнера:
#   echo "Hello from container" > /data/test.txt
#   exit

# 4. Запуск второго контейнера с тем же томом
sudo docker run -it --name volume-test2 -v my-data:/data ubuntu:22.04 bash
# Внутри контейнера:
#   cat /data/test.txt
#   exit

# 5. Удаление тома (после удаления контейнеров)
sudo docker volume rm my-data
```

### Задание 6: Использование Docker Compose

```bash
# 1. Создание docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html
    networks:
      - my-network

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: mydb
      MYSQL_USER: user
      MYSQL_PASSWORD: userpass
    networks:
      - my-network
    volumes:
      - mysql-data:/var/lib/mysql

volumes:
  mysql-data:

networks:
  my-network:
    driver: bridge
EOF

# 2. Создание HTML-страницы
mkdir html
echo "<h1>Hello from Docker Compose!</h1>" > html/index.html

# 3. Запуск сервисов
sudo docker compose up -d

# 4. Просмотр статуса
sudo docker compose ps

# 5. Просмотр логов
sudo docker compose logs

# 6. Остановка сервисов
sudo docker compose down
```

---

## Контрольные вопросы

1. **Основные понятия:**
   - В чем разница между образом и контейнером?
   - Что такое Dockerfile и для чего он используется?
   - Что такое Docker Hub?

2. **Команды Docker:**
   - Как просмотреть все запущенные контейнеры?
   - Как удалить все неиспользуемые образы?
   - Как подключиться к запущенному контейнеру?

3. **Практические задачи:**
   - Как перенести данные между хостом и контейнером?
   - Как сделать так, чтобы контейнер запускался автоматически при старте системы?
   - Как ограничить использование ресурсов контейнера?

---

## Дополнительные задания для самостоятельной работы

1. Создайте Dockerfile для Node.js приложения
2. Настройте многоступенчатую сборку (multi-stage build)
3. Создайте сеть Docker и подключите к ней несколько контейнеров
4. Настройте автоматическую пересборку при изменении кода (watch mode)
5. Изучите Docker Swarm для оркестрации контейнеров

---

## Полезные команды для очистки

```bash
# Удаление всех остановленных контейнеров
sudo docker container prune

# Удаление всех неиспользуемых образов
sudo docker image prune -a

# Удаление всех неиспользуемых томов
sudo docker volume prune

# Полная очистка системы Docker
sudo docker system prune -a
```

## Критерии оценки

- ✅ Установлен Docker и выполнены базовые команды
- ✅ Создан и запущен собственный образ из Dockerfile
- ✅ Работа с томами и сетями
- ✅ Использование Docker Compose для многоконтейнерных приложений
- ✅ Понимание основных концепций Docker

---

## Ресурсы для дальнейшего изучения

1. [Официальная документация Docker](https://docs.docker.com/)
2. [Docker Hub](https://hub.docker.com/)
3. [Best practices для написания Dockerfile](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
4. [Примеры Dockerfile для разных языков](https://github.com/docker-library/docs)

**Примечание:** Все команды выполняются с `sudo` или от пользователя, добавленного в группу `docker`. Для учебных целей рекомендуется использовать виртуальную машину или выделенный сервер.