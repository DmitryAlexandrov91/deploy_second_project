<div align="center">
<h1> Taski Docker </h1>
<p><em> Учебный проект для деплоя на сервер при помощи Ci/CD</em></p>
</div>

▌ Описание ⚙️


Проект представляет собой пример деплоя приложения, состоящего из backend и frontend частей, при помощи CI/CD.

В проекте рассматриваются:

- основы "упаковывания" приложений в контейнеры
- создание "оркестров" на основе образов и Dockerfile
- настройка взаимосвязей между контейнерами внутри "оркестра"
- основы автоматизации деплоя при помощи workflows


---
▌ Установка и запуск 🛠️

Переда началом работы нужно установить себе на ПК или на сервер Docker(иструкцию можно найти в проекте "deploy_first_project")

Склонируйте себе репозиторий проекта, установите виртуальное окружение и установите зависимости из директории backend:

```bash
python3.9 -m venv venv  # Linux | Mac Os
python -3.9 -m venv venv  # Windows

source venv/bin/avtivate  # # Linux | Mac Os
. venv/scripts/activate  # Windows

cd /backend
pip install -r requirements.txt
```

### Локальный запуск

Для локального запуска проекта подготовлен файл с оркестром docker-compose.yml<br>
"Оркестр" представляет собой инструкцию, которая запускает контейнеры на основе образов и "рецептов" Dockerfile.<br>
Инструкции пишутся на языке YAML, это родственник JSON и XML. В настоящее время его в основном применяют для хранения структурированных данных в текстовых файлах.

Убедитесь, что у вас запущен Docker и выполните команду 

```bash
docker compose up  # с ключом -d в конце чтобы "освободить" терминал после запуска контейнера
```

Что сделает Docker Compose "под капотом":

- получит готовые образы, указанные в image,
- соберёт все образы, указанные в build,
- запустит все контейнеры, описанные в конфиге.

Почитайте комментарии в docker-compose.yml для понимания конкретных шагов.
Обратите внимание, что контейнер с фронтом останавливается после старта. Это нормально, т.к. всё необходимое он сделал - скопировал файлы статики в volume под названием static.

Полезные команды упраления контейнерами:
```bash
docker compose stop  # остановка оркестра
docker compose up --build  # старт оркестра с пересборкой контейнеров

```

Теперь примените миграции в контейнере с бэкендом

```bash
docker compose exec backend python manage.py migrate 
```

Соберите статику Django приложения и копируйте её в volume static

```bash
docker compose exec backend python manage.py collectstatic
# Статика приложения в контейнере backend 
# будет собрана в директорию /app/collected_static/.

# Теперь из этой директории копируем статику в /backend_static/static/;
# эта статика попадёт на volume static в папку /static/:
docker compose exec backend cp -r /app/collected_static/. /backend_static/static/
```

Приложение запущено, статика раздаётся. 
Главная страница доступна по адресу *http://localhost:9000/*
Админка *http://localhost:9000/admin/*

### Запуск на сервере

Для запуска проекта на сервере воспользуемся docker-compose.production.yml<br>
Вся разница - контейнеры будут собираться из подготовленных образов с вашего Docker Hub

#### Перед началом деплоя проекта на сервере проведите подготовительные процедуры локально.

```bash
docker compose down  # удаляет все контейнеры и связанные сети

docker compose down -v  #  удаляет все контейнеры, связанные сети и volumes
```

Соберите образы:
```bash
cd frontend  # В директории frontend...
docker build -t username/taski_frontend .  # ...сбилдить образ, назвать его taski_frontend
cd ../backend  # То же в директории backend...
docker build -t username/taski_backend .
cd ../gateway  # ...то же и в gateway
docker build -t username/taski_gateway . 
```

Проверьте, что на вашем компьютере созданы необходимые образы
```bash
docker image ls
```

Загрузите образы на Docker Hub:

- если у вас Mac на Apple Silicon (M1/M2)

```bash
# На вашем компьютере образы собираются под архитектуру процессора ARM вашего Apple Silicon, а на сервере будет другой процессор, 64-битный x86-совместимый. Эти архитектуры обозначаются по-разному: ARM64 и AMD64. Команды процессоров отличаются, и образ просто не запустится.
# Чтобы адаптировать образы под x86, их нужно собирать при помощи утилиты Docker BuildX. Она уже есть на вашем компьютере — она была установлена вместе с Docker. 
# Docker BuildX — это специальная обёртка над более сложным сборщиком образов Docker, BuildKit. Он позволяет собирать образы для других платформ. 
# Для сборки образов перейдите в папку taski/ и выполните команды из листинга:

# В директории backend...
cd backend
# ...билдим образы для архитектуры AMD64 
docker buildx build --platform=linux/amd64 -t username/taski_backend .
cd ../frontend
docker buildx build --platform=linux/amd64 -t username/taski_frontend .
cd ../gateway
docker buildx build --platform=linux/amd64 -t username/taski_gateway .
# Отправляем на Docker Hub
docker push username/taski_backend
docker push username/taski_frontend
docker push username/taski_gateway 

# При сборке образов BuildX позволяет указать целевую архитектуру процессора, и за счёт этого можно собрать образы, которые можно будет запускать и на процессорах Intel/AMD.
# Образы, собранные на Apple Silicon без использования BuildX, не смогут запуститься на удалённом сервере другой архитектуры.
```
- иначе

```bash
docker push username/taski_frontend
docker push username/taski_backend
docker push username/taski_gateway 
```

Запустите Docker Compose с этой конфигурацией на своём компьютере.<br>
Название файла конфигурации надо указать явным образом, ведь оно отличается от дефолтного. Имя файла указывается после ключа -f.

```bash
docker compose -f docker-compose.production.yml up -d  # c ключом -d чтобы не открывать новый терминал
```

Сразу же соберите статику:
```bash
docker compose -f docker-compose.production.yml exec backend python manage.py collectstatic
docker compose -f docker-compose.production.yml exec backend cp -r /app/collected_static/. /backend_static/static/
```

Примените миграции:
```bash
docker compose -f docker-compose.production.yml exec backend python manage.py migrate
```

Проверьте, что страница http://localhost:8000/api/tasks/ заработала. 
Если всё хорошо — остановите Docker Compose командой
```bash
docker compose -f docker-compose.production.yml stop
```
Если что-то пошло не так — проверьте все шаги выше.

#### Подготовительные процедуры на сервере

Подключитесь к серверу
```bash
# Используйте параметр -i, 
# если файл с SSH-ключом называется не .ssh/id_rsa, а иначе:
ssh -i путь_до_файла_с_SSH_ключом/название_файла_с_SSH_ключом имя_пользователя@ip_адрес_сервера
```

Остановите Gunicorn:
```bash
sudo systemctl stop gunicorn 
```

Удалите юнит gunicorn, чтобы он больше не перезапускался:
```bash
sudo rm /etc/systemd/system/gunicorn.service 
```

Удалите лишние директории с ненужными проектами на сервере:
```bash
# Удалить папку со всем содержимым
rm -rf /лишняя_папка
# Удалить содержимое папки, но не саму эту папку:
rm -r ./лишняя_папка/* 
```
Создайте директорию /taski
```bash
mkdir taski
```

Почистите диск сервера

```bash
npm cache clean --force  # очистка кеша npm,в нём содержатся сохранённые файлы зависимостей фронтенда, которые обычно требуются, чтобы не скачивать их повторно.

sudo apt clean  # очистка кеша APT: он хранит файлы для установки системных зависимостей Linux

sudo journalctl --vacuum-time=1d  # удаление старых системных логов
``` 

Установите Docker Compose на сервер (если установлен, пропутите шаг)

```bash
sudo apt update
sudo apt install curl
curl -fSL https://get.docker.com -o get-docker.sh
sudo sh ./get-docker.sh
sudo apt install docker-compose-plugin
```

Запустите Docker Compose на сервере

Скопируйте на сервер в директорию taski/ файл docker-compose.production.yml

**Вариант 1 через утилиту SCP:**<br>
Зайдите на своём компьютере в директорию taski/ и выполните команду копирования:
```bash
scp -i path_to_SSH/SSH_name docker-compose.production.yml \
    username@server_ip:/home/username/taski/docker-compose.production.yml
```

- path_to_SSH — путь к файлу с SSH-ключом;
- SSH_name — имя файла с SSH-ключом (без расширения);
- username — ваше имя пользователя на сервере;
- server_ip — IP вашего сервера.

**Вариант 2 методом копировать вставить**
Создайте на сервере пустой файл docker-compose.production.yml и с помощью редактора nano добавьте в него содержимое из локального docker-compose.production.yml. 

Скопируйте файл .env на сервер, в директорию taski/.  (пример в файле .env.example)

Запустите "оркестр" на сервере

```bash
sudo docker compose -f docker-compose.production.yml up -d 
```

Основные команды, которые вам понадобятся для управления:
```bash
docker compose stop  # остановит все контейнеры, но оставит сети и volume. Эта команда пригодится, чтобы перезагрузить или обновить приложения
docker compose down  # остановит все контейнеры, удалит их, сети и анонимные volumes. Можно будет начать всё заново
docker compose logs  # просмотр логов запущенных контейнеров.
```

Проверьте, что все нужные контейнеры запущены:
```bash
sudo docker compose -f docker-compose.production.yml ps 
```

Выполните миграции, соберите статические файлы бэкенда и скопируйте их в /backend_static/static/:

```bash
sudo docker compose -f docker-compose.production.yml exec backend python manage.py migrate
sudo docker compose -f docker-compose.production.yml exec backend python manage.py collectstatic
sudo docker compose -f docker-compose.production.yml exec backend cp -r /app/collected_static/. /backend_static/static/
```

Чтобы на сервере заработала статика нужно перенаправить все запросы в докер

Находясь на сервере, откройте конфиг Nginx:
```bash
sudo nano /etc/nginx/sites-enabled/default
```

Проверьте настройки location в секции server.
Должен быть один блок location:
```bash
location / {
        proxy_set_header Host $http_host;
        proxy_pass http://127.0.0.1:9090;
    }
```

Чтобы убедиться, что в конфиге нет ошибок — выполните команду проверки конфигурации:
```bash
sudo nginx -t 
```

Если всё в порядке, перезагрузите конфиг:
```bash
sudo service nginx reload 
```


▌ Автор 📝

Александров Дмитрий

<u>GitHub</u>
- https://github.com/DmitryAlexandrov91

<u>Telegram</u>
- https://t.me/@AlDmAl

<u>Habr Career</u>
- https://career.habr.com/aldmal