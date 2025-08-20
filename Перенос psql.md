# Перенос PostgreSQL базы данных с локального компьютера на сервер

## 1. Создание дампа на локальном компьютере

### Полный дамп базы данных

```bash

# Дамп с дополнительными опциями (рекомендуется)
pg_dump -U username -h localhost -p 5432 -Fc --no-owner --no-privileges database_name > database_dump.dump

# Только структура (без данных)
pg_dump -U username -h localhost -p 5432 --schema-only database_name > structure_only.sql

# Только данные (без структуры)
pg_dump -U username -h localhost -p 5432 --data-only database_name > data_only.sql
```

**Объяснение параметров:**

- `-U username` - имя пользователя PostgreSQL
- `-h localhost` - хост базы данных
- `-p 5432` - порт PostgreSQL
- `-Fc` - формат custom (сжатый, быстрее восстанавливается)
- `--no-owner` - не включать команды смены владельца
- `--no-privileges` - не включать команды прав доступа

## 2. Установка PostgreSQL на сервере Ubuntu

```bash
# Обновление пакетов
sudo apt update

# Установка PostgreSQL
sudo apt install postgresql postgresql-contrib -y

# Проверка версии
psql --version

# Запуск службы
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

## 3. Настройка PostgreSQL на сервере

### 3.1 Изменение метода аутентификации с peer на md5

```bash
# Открываем файл конфигурации
sudo nano /etc/postgresql/*/main/pg_hba.conf
```

**Найдите и измените строки:**

```
# БЫЛО:
local   all             all                                     peer
host    all             all             127.0.0.1/32            ident

# СТАЛО:
local   all             all                                     md5
host    all             all             127.0.0.1/32            md5
host    all             all             0.0.0.0/0               md5
```

### 3.2 Настройка для внешних подключений (опционально)

```bash
# Открываем основной файл конфигурации
sudo nano /etc/postgresql/*/main/postgresql.conf
```

**Найдите и измените:**

```
# БЫЛО:
#listen_addresses = 'localhost'

# СТАЛО:
listen_addresses = '*'
```

### 3.3 Перезапуск PostgreSQL

```bash
sudo systemctl restart postgresql
```

## 4. Создание пользователя и базы данных

### 4.1 Подключение к PostgreSQL

```bash
# Переключение на пользователя postgres
sudo -u postgres psql
```

### 4.2 Создание пользователя

```sql
-- Создание пользователя с паролем
CREATE USER your_username WITH PASSWORD 'your_password';

-- Предоставление прав суперпользователя (если нужно)
ALTER USER your_username WITH SUPERUSER;

-- Или предоставление конкретных прав
ALTER USER your_username WITH CREATEDB;
ALTER USER your_username WITH LOGIN;

-- Выход из psql
\q
```

### 4.3 Создание базы данных

```bash
# Создание базы данных от имени нового пользователя
sudo -u postgres createdb -O your_username your_database_name

# Или через psql
sudo -u postgres psql -c "CREATE DATABASE your_database_name OWNER your_username;"
```

## 5. Перенос дампа на сервер

### 5.1 Копирование файла на сервер

```bash
# Через scp
scp database_dump.sql root@ip:/tmp/

# Через rsync
rsync -avz database_dump.sql root@ip:/tmp/

# Или загрузить через SFTP клиент
```

## 6. Восстановление базы данных

### 6.1 Проверка подключения

```bash
# Проверка подключения к базе
psql -U your_username -d your_database_name -h localhost
```

### 6.2 Очистка базы данных (если нужно)

```bash
# Подключение к базе
psql -U your_username -d your_database_name -h localhost

# В psql выполните:
```

```sql
-- Отключение всех подключений к базе
SELECT pg_terminate_backend(pg_stat_activity.pid)
FROM pg_stat_activity
WHERE pg_stat_activity.datname = 'your_database_name'
  AND pid <> pg_backend_pid();

-- Удаление всех таблиц
DROP SCHEMA public CASCADE;
CREATE SCHEMA public;
GRANT ALL ON SCHEMA public TO your_username;
GRANT ALL ON SCHEMA public TO public;

-- Выход
\q
```

### 6.3 Восстановление из дампа

```bash
# Для SQL дампа
psql -U your_username -d your_database_name -h localhost < /tmp/database_dump.sql

# Для custom формата (.dump)
pg_restore -U your_username -d your_database_name -h localhost -v /tmp/database_dump.dump

# С дополнительными опциями
pg_restore -U your_username -d your_database_name -h localhost -v --clean --no-owner --no-privileges /tmp/database_dump.dump
```

**Параметры pg_restore:**

- `-v` - подробный вывод
- `--clean` - очистить существующие объекты
- `--no-owner` - не восстанавливать владельцев
- `--no-privileges` - не восстанавливать права доступа

## 7. Проверка восстановления

```bash
# Подключение к базе
psql -U your_username -d your_database_name -h localhost

# Проверка таблиц
\dt

# Проверка количества записей в таблицах
SELECT schemaname,tablename,n_tup_ins,n_tup_upd,n_tup_del,n_live_tup,n_dead_tup 
FROM pg_stat_user_tables;

# Проверка размера базы
SELECT pg_size_pretty(pg_database_size('your_database_name'));

# Выход
\q
```

## 8. Настройка прав доступа

```sql
-- Подключение к базе как суперпользователь
sudo -u postgres psql -d your_database_name

-- Предоставление всех прав на все таблицы
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO your_username;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO your_username;
GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA public TO your_username;

-- Предоставление прав на будущие объекты
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO your_username;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO your_username;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON FUNCTIONS TO your_username;
```

## 9. Настройка файрвола (если нужен внешний доступ)

```bash
# Разрешить подключения к PostgreSQL
sudo ufw allow 5432

# Или только с конкретного IP
sudo ufw allow from your_ip_address to any port 5432
```

## 10. Полезные команды для отладки

```bash
# Просмотр логов PostgreSQL
sudo tail -f /var/log/postgresql/postgresql-*-main.log

# Проверка статуса службы
sudo systemctl status postgresql

# Перезапуск службы
sudo systemctl restart postgresql

# Проверка активных подключений
sudo -u postgres psql -c "SELECT * FROM pg_stat_activity;"

# Проверка размера всех баз
sudo -u postgres psql -c "SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database;"
```

## 11. Пример полного скрипта переноса

```bash
#!/bin/bash

# Переменные
DB_NAME="database"
DB_USER="password"
DB_PASS="strong_password"
DUMP_FILE="/tmp/database_dump.sql"

echo "=== Создание пользователя и базы данных ==="
sudo -u postgres psql -c "CREATE USER $DB_USER WITH PASSWORD '$DB_PASS';"
sudo -u postgres psql -c "ALTER USER $DB_USER WITH CREATEDB;"
sudo -u postgres psql -c "CREATE DATABASE $DB_NAME OWNER $DB_USER;"

echo "=== Восстановление дампа ==="
psql -U $DB_USER -d $DB_NAME -h localhost < $DUMP_FILE

echo "=== Настройка прав ==="
sudo -u postgres psql -d $DB_NAME -c "GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO $DB_USER;"
sudo -u postgres psql -d $DB_NAME -c "GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO $DB_USER;"

echo "=== Проверка ==="
psql -U $DB_USER -d $DB_NAME -h localhost -c "\dt"

echo "Готово!"
```

## Часто встречающиеся ошибки

### Ошибка: "peer authentication failed"

**Решение:** Проверьте настройки в `/etc/postgresql/*/main/pg_hba.conf` - должно быть `md5` вместо `peer`

### Ошибка: "database does not exist"

**Решение:** Создайте базу данных перед восстановлением дампа

### Ошибка: "permission denied"

**Решение:** Настройте права доступа для пользователя

### Ошибка: "could not connect to server"

**Решение:** Проверьте что PostgreSQL запущен и слушает нужный порт

## Заключение

После выполнения всех шагов ваша база данных будет полностью перенесена на сервер. Не забудьте обновить строки подключения в вашем приложении для работы с новой базой данных.