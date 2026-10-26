# Lesson 17 — Auto Orders API: развёртывание на Ubuntu

Полная инструкция по развёртыванию приложения Auto Orders API (FastAPI + PostgreSQL + psycopg3) на Ubuntu как systemd-сервиса, с Nginx в роли reverse proxy и ежедневными бэкапами PostgreSQL.

Общие шаги (systemd, Nginx, бэкап) пересекаются с lesson 16, но здесь приведены целиком — документ автономен.

---

## Исходные данные

| Параметр | Значение |
|---|---|
| Точка входа | main.py |
| Порт приложения | 8000 |
| Каталог приложения | /opt/carss |
| Пользователь/группа | webadmin:webgroup |
| Имя systemd-сервиса | app-carss.service |
| БД | autos |
| Пользователь БД | carss_user |
| Пароль БД | StrongPassw0rd! |

Стек: Python 3.12+, FastAPI, Uvicorn, PostgreSQL, psycopg3.

Эндпоинты:

| Метод | Путь | Описание |
|---|---|---|
| GET | /health | Проверка соединения с БД |
| GET | /models | Список моделей авто |
| GET | /orders | Список заказов |

При старте приложение само создаёт таблицы car_models и orders (если их нет) и заполняет car_models начальными данными через init_db() в lifespan. Миграций нет.

---

## Часть 0. Подготовка пользователя и каталогов

### 0.1. Создать системного пользователя и группу

sudo groupadd --system webgroup 2>/dev/null || true
sudo useradd --system --gid webgroup --create-home --shell /usr/sbin/nologin webadmin 2>/dev/null || true

Проверка:

id webadmin

Ожидаемо: uid=... (webadmin) gid=... (webgroup) groups=... (webgroup).

### 0.2. Создать каталог приложения

sudo mkdir -p /opt/carss
sudo chown -R webadmin:webgroup /opt/carss

### 0.3. Скопировать файлы проекта

Скопируйте main.py и requirements.txt из репозитория в /opt/carss/:

sudo cp main.py requirements.txt /opt/carss/
sudo chown -R webadmin:webgroup /opt/carss

Проверка:

ls -l /opt/carss/

Должны быть main.py, requirements.txt (и, возможно, README.md).

---

## Часть 1. Python-окружение

### 1.1. Создать виртуальное окружение

sudo -u webadmin python3 -m venv /opt/carss/venv

Проверка:

/opt/carss/venv/bin/python --version

### 1.2. Установить зависимости

sudo -u webadmin /opt/carss/venv/bin/pip install --upgrade pip
sudo -u webadmin /opt/carss/venv/bin/pip install -r /opt/carss/requirements.txt

Проверка:

/opt/carss/venv/bin/pip list | grep -Ei 'fastapi|uvicorn|psycopg'

Ожидаемо: fastapi, uvicorn, psycopg (и psycopg-binary).

---

## Часть 2. PostgreSQL

### 2.1. Установить PostgreSQL

sudo apt update
sudo apt install -y postgresql

Проверка:

sudo systemctl status postgresql --no-pager
sudo -u postgres psql -c "SELECT version();"

Команда sudo -u postgres psql работает через Unix-сокет (peer-аутентификация), пароль не нужен.

### 2.2. Создать роль carss_user

sudo -u postgres psql -c "CREATE USER carss_user WITH PASSWORD 'StrongPassw0rd!';"

Проверка:

sudo -u postgres psql -c "\du carss_user"

### 2.3. Создать БД autos

sudo -u postgres psql -c "CREATE DATABASE autos OWNER carss_user;"

Проверка:

sudo -u postgres psql -c "\l autos"

### 2.4. Выдать права на БД и схему public

Права на БД:

sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE autos TO carss_user;"

Обязательно для PG 15+ — иначе init_db() упадёт с permission denied for schema public:

sudo -u postgres psql -d autos -c "ALTER SCHEMA public OWNER TO carss_user;"
sudo -u postgres psql -d autos -c "GRANT ALL ON SCHEMA public TO carss_user;"

### 2.5. Проверить TCP-подключение от carss_user

psql 'postgresql://carss_user:StrongPassw0rd!@localhost:5432/autos' -c "SELECT current_user, current_database();"

Ожидаемо: carss_user | autos.

Проверка права создавать таблицы:

psql 'postgresql://carss_user:StrongPassw0rd!@localhost:5432/autos' -c "CREATE TABLE _t(id int);"
psql 'postgresql://carss_user:StrongPassw0rd!@localhost:5432/autos' -c "DROP TABLE _t;"

Важно: строку подключения берите в одинарные кавычки — иначе ! в пароле вызовет event not found в интерактивном bash. Если хотите избежать сюрпризов, смените пароль на такой, где нет !, $, \, обратных кавычек, ".

### 2.6. Если пароль ранее был другим — сбросить

sudo -u postgres psql -c "ALTER USER carss_user WITH PASSWORD 'StrongPassw0rd!';"

---

## Часть 3. Переменные окружения и systemd-сервис

### 3.1. Создать /etc/carss.env

sudo tee /etc/carss.env >/dev/null <<'EOF'
DATABASE_URL=postgresql://carss_user:StrongPassw0rd!@localhost:5432/autos
EOF
sudo chown root:webgroup /etc/carss.env
sudo chmod 640 /etc/carss.env

Проверка, что webadmin может прочитать файл:

sudo -u webadmin cat /etc/carss.env

Если Permission denied — убедитесь, что webadmin в группе webgroup:

id webadmin
sudo usermod -aG webgroup webadmin

После usermod перезапустите сервис — systemd подхватит группу при следующем старте процесса.

### 3.2. Создать unit-файл /etc/systemd/system/app-carss.service

sudo tee /etc/systemd/system/app-carss.service >/dev/null <<'EOF'
[Unit]
Description=app-carss
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=simple
User=webadmin
Group=webgroup
WorkingDirectory=/opt/carss
EnvironmentFile=/etc/carss.env
ExecStart=/opt/carss/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

Флаг --host 0.0.0.0 — приложение слушает все интерфейсы. Если Nginx на той же машине, безопаснее --host 127.0.0.1 и proxy_pass http://127.0.0.1:8000; в Nginx. Но с 0.0.0.0 тоже работает.

### 3.3. Активировать и запустить сервис

sudo systemctl daemon-reload
sudo systemctl enable --now app-carss.service

Проверка:

sudo systemctl status app-carss.service --no-pager

Ожидаемо: Active: active (running).

### 3.4. Проверить API изнутри

curl -s http://127.0.0.1:8000/health
curl -s http://127.0.0.1:8000/models
curl -s http://127.0.0.1:8000/orders

Ожидаемо: {"status":"ok"}, три модели (Tesla Model 3, BMW X5, Toyota Camry), [] для заказов.

### 3.5. Если сервис падает — диагностика

Посмотреть полный traceback:

sudo journalctl -u app-carss.service -n 50 --no-pager

Запустить вручную от webadmin, чтобы увидеть ошибку вживую:

sudo -u webadmin /opt/carss/venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000

Типовые причины падения (код выхода 3 = STARTUP_FAILURE):

| Симптом в логе | Причина | Решение |
|---|---|---|
| password authentication failed for user "postgres" | EnvironmentFile не читается, main.py берёт дефолт postgres:postgres | Проверить EnvironmentFile в unit, права /etc/carss.env, группу webadmin |
| password authentication failed for user "carss_user" | Пароль в env не совпадает с паролем в БД | ALTER USER carss_user WITH PASSWORD ..., синхронизировать env |
| database "autos" does not exist | БД не создана или называется иначе | Создать БД autos (Часть 2.3) |
| permission denied for schema public | Нет прав на схему | Часть 2.4 |
| ModuleNotFoundError: No module named 'main' | WorkingDirectory не /opt/carss или main.py не там | Проверить WorkingDirectory и ls /opt/carss |

Сбросить счётчик рестартов перед повторным запуском:

sudo systemctl reset-failed app-carss.service

---

## Часть 4. Nginx как reverse proxy

### 4.1. Установить Nginx

sudo apt install -y nginx

Проверка:

sudo systemctl status nginx --no-pager

### 4.2. Создать конфиг /etc/nginx/sites-available/carss

sudo tee /etc/nginx/sites-available/carss >/dev/null <<'EOF'
server {
    listen 80;
    server_name _;   # замените на свой домен, если есть

    access_log /var/log/nginx/carss.access.log;
    error_log  /var/log/nginx/carss.error.log;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
    }
}
EOF

Если в unit-файле --host 0.0.0.0, proxy_pass на 127.0.0.1:8000 всё равно работает. Если хотите строже — поменяйте в unit на --host 127.0.0.1.

### 4.3. Активировать сайт и убрать дефолтный

sudo ln -sf /etc/nginx/sites-available/carss /etc/nginx/sites-enabled/carss
sudo rm -f /etc/nginx/sites-enabled/default

### 4.4. Проверить конфиг и перезагрузить

sudo nginx -t

Ожидаемо: syntax is ok, test is successful.

sudo systemctl reload nginx

### 4.5. Проверить API через Nginx

curl -s http://127.0.0.1/health
curl -s http://IP-сервера/health

Ожидаемо: {"status":"ok"}.

Диагностика:

| Симптом | Причина | Решение |
|---|---|---|
| 502 Bad Gateway | Uvicorn не слушает порт | sudo ss -ltnp | grep 8000, проверить systemctl status app-carss |
| 403 Forbidden | Нет index, autoindex off | Это нормально для / без эндпоинта; проверьте /health |
| Nginx не стартует | Синтаксис | sudo nginx -t покажет строку |

---

## Часть 5. Скрипт бэкапа БД

### 5.1. Настроить .pgpass для безпарольного pg_dump

sudo -u webadmin tee /home/webadmin/.pgpass >/dev/null <<'EOF'
localhost:5432:autos:carss_user:StrongPassw0rd!
EOF
sudo chown webadmin:webgroup /home/webadmin/.pgpass
sudo chmod 600 /home/webadmin/.pgpass

Проверка, что pg_dump работает без пароля:

sudo -u webadmin pg_dump -h localhost -U carss_user -d autos --no-owner --no-privileges | head -5

Символ ! в .pgpass не интерпретируется shell'ом — файл читает сам libpq.

### 5.2. Создать каталог для бэкапов

sudo mkdir -p /var/backups/carss
sudo chown webadmin:webgroup /var/backups/carss
sudo chmod 750 /var/backups/carss

### 5.3. Создать скрипт /usr/local/bin/backup_carss.sh

sudo tee /usr/local/bin/backup_carss.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
# Бэкап БД Auto Orders API (PostgreSQL)
set -euo pipefail

BACKUP_DIR="/var/backups/carss"
DB_NAME="autos"
DB_USER="carss_user"
DB_HOST="localhost"
DB_PORT="5432"
RETENTION_DAYS=7
DATE=$(date +%Y%m%d_%H%M%S)
FILE="${BACKUP_DIR}/${DB_NAME}_${DATE}.sql.gz"

mkdir -p "$BACKUP_DIR"

# Бэкап: gzip-сжатый SQL-дамп
pg_dump -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" \
        --no-owner --no-privileges \
    | gzip > "$FILE"

# Проверка, что файл не пустой
if [ ! -s "$FILE" ]; then
    echo "ERROR: backup file is empty: $FILE" >&2
    exit 1
fi

# Проверка целостности gzip
gzip -t "$FILE"

# Удаление старых дампов
find "$BACKUP_DIR" -name "${DB_NAME}_*.sql.gz" -mtime +"$RETENTION_DAYS" -delete

echo "[$(date -Is)] backup OK: $FILE ($(du -h "$FILE" | cut -f1))"
EOF
sudo chmod 750 /usr/local/bin/backup_carss.sh
sudo chown root:webgroup /usr/local/bin/backup_carss.sh

### 5.4. Первый ручной запуск

sudo -u webadmin /usr/local/bin/backup_carss.sh

Проверка:

ls -lh /var/backups/carss/

Ожидаемо: файл autos_YYYYMMDD_HHMMSS.sql.gz размером несколько КБ.

### 5.5. Проверить восстановление (важно!)

Восстанавливаем во временную БД, чтобы не трогать рабочую:

sudo -u postgres createdb autos_restore_test
gunzip -c /var/backups/carss/autos_*.sql.gz | sudo -u postgres psql -d autos_restore_test
sudo -u postgres psql -d autos_restore_test -c "SELECT * FROM car_models;"

Ожидаемо: три модели. Затем удалить тестовую БД:

sudo -u postgres dropdb autos_restore_test

### 5.6. Cron на ежедневный запуск в 02:00

sudo tee /etc/cron.d/carss-backup >/dev/null <<'EOF'
# Ежедневный бэкап Auto Orders API
0 2 * * * webadmin /usr/local/bin/backup_carss.sh >> /var/log/carss-backup.log 2>&1
EOF
sudo touch /var/log/carss-backup.log
sudo chown webadmin:webgroup /var/log/carss-backup.log

Проверка:

sudo systemctl status cron --no-pager | head
sudo cat /var/log/carss-backup.log

---

## Часть 6. Финальная проверка

### 6.1. Сервис

systemctl is-active app-carss.service

Ожидаемо: active.

systemctl is-enabled app-carss.service

Ожидаемо: enabled.

### 6.2. API через Nginx

curl -s http://127.0.0.1/health
curl -s http://127.0.0.1/models
curl -s http://127.0.0.1/orders

### 6.3. Бэкап

sudo -u webadmin /usr/local/bin/backup_carss.sh
ls -lh /var/backups/carss/

### 6.4. Cron

sudo systemctl status cron --no-pager | head
sudo cat /var/log/carss-backup.log

---

## Часть 7. Шпаргалка «с нуля»

Собранный последовательный скрипт (запускать по частям, проверяя вывод):

# 0. Пользователь и каталоги
sudo groupadd --system webgroup 2>/dev/null || true
sudo useradd --system --gid webgroup --create-home --shell /usr/sbin/nologin webadmin 2>/dev/null || true
sudo mkdir -p /opt/carss
sudo chown -R webadmin:webgroup /opt/carss
sudo cp main.py requirements.txt /opt/carss/
sudo chown -R webadmin:webgroup /opt/carss

# 1. venv
sudo -u webadmin python3 -m venv /opt/carss/venv
sudo -u webadmin /opt/carss/venv/bin/pip install --upgrade pip
sudo -u webadmin /opt/carss/venv/bin/pip install -r /opt/carss/requirements.txt

# 2. PostgreSQL
sudo apt update && sudo apt install -y postgresql
sudo -u postgres psql -c "CREATE USER carss_user WITH PASSWORD 'StrongPassw0rd!';"
sudo -u postgres psql -c "CREATE DATABASE autos OWNER carss_user;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE autos TO carss_user;"
sudo -u postgres psql -d autos -c "ALTER SCHEMA public OWNER TO carss_user;"
sudo -u postgres psql -d autos -c "GRANT ALL ON SCHEMA public TO carss_user;"
psql 'postgresql://carss_user:StrongPassw0rd!@localhost:5432/autos' -c "SELECT 1;"

# 3. env + systemd
sudo tee /etc/carss.env >/dev/null <<'EOF'
DATABASE_URL=postgresql://carss_user:StrongPassw0rd!@localhost:5432/autos
EOF
sudo chown root:webgroup /etc/carss.env
sudo chmod 640 /etc/carss.env

sudo tee /etc/systemd/system/app-carss.service >/dev/null <<'EOF'
[Unit]
Description=app-carss
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=simple
User=webadmin
Group=webgroup
WorkingDirectory=/opt/carss
EnvironmentFile=/etc/carss.env
ExecStart=/opt/carss/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now app-carss.service
sudo systemctl status app-carss.service --no-pager
curl -s http://127.0.0.1:8000/health

# 4. Nginx
sudo apt install -y nginx
sudo tee /etc/nginx/sites-available/carss >/dev/null <<'EOF'
server {
    listen 80;
    server_name _;
    access_log /var/log/nginx/carss.access.log;
    error_log  /var/log/nginx/carss.error.log;
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
    }
}
EOF
sudo ln -sf /etc/nginx/sites-available/carss /etc/nginx/sites-enabled/carss
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
curl -s http://127.0.0.1/health

# 5. Бэкап
sudo -u webadmin tee /home/webadmin/.pgpass >/dev/null <<'EOF'
localhost:5432:autos:carss_user:StrongPassw0rd!
EOF
sudo chown webadmin:webgroup /home/webadmin/.pgpass
sudo chmod 600 /home/webadmin/.pgpass
sudo mkdir -p /var/backups/carss
sudo chown webadmin:webgroup /var/backups/carss
sudo chmod 750 /var/backups/carss
# (создать /usr/local/bin/backup_carss.sh, см. Часть 5.3)
sudo chmod 750 /usr/local/bin/backup_carss.sh
sudo chown root:webgroup /usr/local/bin/backup_carss.sh
sudo -u webadmin /usr/local/bin/backup_carss.sh

sudo tee /etc/cron.d/carss-backup >/dev/null <<'EOF'
0 2 * * * webadmin /usr/local/bin/backup_carss.sh >> /var/log/carss-backup.log 2>&1
EOF
sudo touch /var/log/carss-backup.log
sudo chown webadmin:webgroup /var/log/carss-backup.log

---

## Часть 8. Типовые ошибки

| Ошибка | Причина | Решение |
|---|---|---|
| bash: !@localhost: event not found | ! в двойных кавычках интерактивного bash | Строку подключения в одинарные кавычки: psql 'postgresql://...!...' |
| password authentication failed for user "postgres" в логах сервиса | EnvironmentFile не читается → main.py берёт дефолт postgres:postgres | Проверить EnvironmentFile в unit, права /etc/carss.env, группу webadmin |
| password authentication failed for user "carss_user" | Пароль в env не совпадает с паролем в БД | ALTER USER carss_user WITH PASSWORD ..., синхронизировать /etc/carss.env |
| database "autos" does not exist | БД не создана / другое имя в env | Создать autos, поправить env |
| permission denied for schema public | Нет прав на схему (PG 15+) | ALTER SCHEMA public OWNER TO carss_user; GRANT ALL ON SCHEMA public TO carss_user; |
| role "carss_user" already exists | Повторный CREATE USER | Ничего не делать или DROP ROLE IF EXISTS перед созданием |
| ERROR: Application startup failed. Exiting. + код 3 | Любая ошибка в lifespan (init_db) | Смотреть journalctl -u app-carss -n 50; запустить вручную от webadmin |
| Start request repeated too quickly | Restart=on-failure + падение при старте | sudo systemctl reset-failed app-carss перед рестартом |
| Nginx 502 Bad Gateway | Uvicorn не слушает порт | sudo ss -ltnp | grep 8000; systemctl status app-carss |
| pg_dump: error: connection ... password | .pgpass отсутствует / неверные права | Владелец webadmin, права 600, путь /home/webadmin/.pgpass |
| backup file is empty | pg_dump упал (нет прав / нет БД) | Проверить pg_dump вручную; pg_isready |

---

## Часть 9. Полезные команды

# Логи сервиса
sudo journalctl -u app-carss.service -f

# Перезапуск
sudo systemctl restart app-carss.service

# Сброс счётчика рестартов
sudo systemctl reset-failed app-carss.service

# Логи Nginx
sudo tail -f /var/log/nginx/carss.access.log /var/log/nginx/carss.error.log

# Ручной бэкап
sudo -u webadmin /usr/local/bin/backup_carss.sh

# Список бэкапов
ls -lh /var/backups/carss/

# Восстановление из бэкапа (осторожно — перезапишет данные!)
# gunzip -c /var/backups/carss/autos_YYYYMMDD_HHMMSS.sql.gz | sudo -u postgres psql -d autos_restore

# Проверка портов
sudo ss -ltnp | grep -E ':80|:8000|:5432'

# Проверка БД
psql 'postgresql://carss_user:StrongPassw0rd!@localhost:5432/autos' -c "\dt"

---

## Структура после развёртывания

/opt/carss/
  main.py
  requirements.txt
  README.md
  venv/

/etc/carss.env                          # DATABASE_URL
/etc/systemd/system/app-carss.service   # systemd unit
/etc/nginx/sites-available/carss        # Nginx vhost
/usr/local/bin/backup_carss.sh          # скрипт бэкапа
/etc/cron.d/carss-backup                # cron-задание
/var/backups/carss/                     # каталог бэкапов
/var/log/carss-backup.log               # лог бэкапов

## Переменные окружения

| Переменная | Значение | Где задаётся |
|---|---|---|
| DATABASE_URL | postgresql://carss_user:StrongPassw0rd!@localhost:5432/autos | /etc/carss.env |

## Схема БД

Приложение создаёт таблицы автоматически при старте (init_db() в lifespan):

- car_models — модели авто (id, name, price);
- orders — заказы (id, model_id, customer_name, created_at).

Если car_models пустая — приложение добавляет три начальные модели (Tesla Model 3, BMW X5, Toyota Camry).
