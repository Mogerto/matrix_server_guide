# Полное руководство по установке Matrix-сервера

## Содержание

1. [Введение](#введение)
2. [Требования](#требования)
3. [Подготовка сервера](#подготовка-сервера)
4. [Установка Docker](#установка-docker)
5. [Настройка проекта](#настройка-проекта)
6. [Конфигурация сервисов](#конфигурация-сервисов)
7. [Запуск и проверка](#запуск-и-проверка)
8. [Создание пользователей](#создание-пользователей)
9. [Диагностика и решение проблем](#диагностика-и-решение-проблем)
10. [Обслуживание](#обслуживание)

---

## Введение

Matrix — это открытый стандарт для децентрализованной коммуникации. Данное руководство поможет вам развернуть собственный Matrix-сервер с полной функциональностью, включая:

- Обмен сообщениями
- Голосовые и видео звонки
- Федерацию с другими серверами
- Автоматические SSL-сертификаты

**Время установки:** 30-60 минут  
**Уровень сложности:** Средний

---

## Требования

### Технические требования

| Компонент | Минимум | Рекомендуется |
|-----------|---------|---------------|
| RAM | 2 ГБ | 4 ГБ |
| CPU | 1 vCPU | 2 vCPU |
| Диск | 20 ГБ SSD | 50 ГБ SSD |
| ОС | Ubuntu 20.04+ | Ubuntu 22.04 LTS |

### Что понадобится

- **VDS/VPS сервер** (300-400₽/месяц)
- **Доменное имя** (~200₽/год)
- **SSH-доступ** к серверу
- **Базовые навыки** работы с командной строкой

### Настройка DNS

Создайте A-запись в DNS вашего домена:
```
your.domain.com → IP_ADDRESS_СЕРВЕРА
```

---

## Подготовка сервера

### Подключение к серверу

```bash
ssh root@YOUR_SERVER_IP
```

### Обновление системы

```bash
# Обновляем пакеты
sudo apt update && sudo apt upgrade -y

# Устанавливаем необходимые утилиты
sudo apt install curl wget nano htop -y
```

### Настройка файрвола (опционально)

```bash
# Разрешаем необходимые порты
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw allow 3478/tcp  # TURN
sudo ufw allow 3478/udp  # TURN
sudo ufw --force enable
```

---

## Установка Docker

### Установка Docker Engine

```bash
# Устанавливаем Docker
sudo apt install docker.io docker-compose -y

# Добавляем в автозагрузку
sudo systemctl enable --now docker

# Проверяем установку
docker --version
docker-compose --version
```

### Настройка прав пользователя (опционально)

```bash
# Добавляем пользователя в группу docker
sudo usermod -aG docker $USER

# Перелогиниваемся для применения изменений
exit
# Заходим снова через SSH
```

---

## Настройка проекта

### Создание структуры папок

```bash
# Создаем главную папку проекта
sudo mkdir -p /opt/matrix
cd /opt/matrix

# Создаем подпапки для данных
mkdir -p synapse postgres html configs

# Устанавливаем права
sudo chown -R $USER:$USER /opt/matrix
```

### Генерация паролей и секретов

```bash
# Генерируем пароль для базы данных
DB_PASSWORD=$(openssl rand -base64 32)
echo "DB Password: $DB_PASSWORD"

# Генерируем секрет для TURN-сервера
TURN_SECRET=$(openssl rand -hex 64)
echo "TURN Secret: $TURN_SECRET"

# Сохраняем в файл для справки
echo "DB_PASSWORD=$DB_PASSWORD" > .env
echo "TURN_SECRET=$TURN_SECRET" >> .env
```

**⚠️ ВАЖНО:** Сохраните эти пароли в надежном месте!

---

## Конфигурация сервисов

### 1. Docker Compose конфигурация

Создайте файл `docker-compose.yml`:

```yaml
version: '3.8'

services:
  # PostgreSQL база данных
  db:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: synapse
      POSTGRES_PASSWORD: "YOUR_DB_PASSWORD_HERE"  # Замените на сгенерированный
      POSTGRES_DB: synapse
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8 --lc-collate=C --lc-ctype=C"
    volumes:
      - ./postgres:/var/lib/postgresql/data
    networks:
      - matrix_network

  # Matrix Synapse сервер
  synapse:
    image: matrixdotorg/synapse:latest
    restart: unless-stopped
    volumes:
      - ./synapse:/data
    depends_on:
      - db
    ports:
      - "127.0.0.1:8008:8008"
    networks:
      - matrix_network

  # Caddy reverse proxy
  caddy:
    image: caddy:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./html:/var/www/html
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - matrix_network

  # Coturn TURN сервер для звонков
  coturn:
    image: coturn/coturn:latest
    restart: unless-stopped
    network_mode: "host"
    volumes:
      - ./turnserver.conf:/etc/coturn/turnserver.conf:ro

volumes:
  caddy_data:
  caddy_config:

networks:
  matrix_network:
    driver: bridge
```

### 2. Генерация конфигурации Synapse

**Важно:** С версии Synapse 1.99+ используется новый репозиторий Element.

```bash
# Замените your.domain.com на ваш домен
sudo docker run -it --rm \
  -v ./synapse:/data \
  -e SYNAPSE_SERVER_NAME=your.domain.com \
  -e SYNAPSE_REPORT_STATS=yes \
  matrixdotorg/synapse:latest generate

# Альтернативная команда через docker-compose
sudo docker-compose run --rm synapse generate \
  --server-name your.domain.com \
  --report-stats=yes \
  --config-path /data/homeserver.yaml
```

**Примечание:** Начиная с версии 1.99, Synapse поддерживается компанией Element под новой лицензией. Функциональность остается той же, но репозиторий изменился с `github.com/matrix-org/synapse` на `github.com/element-hq/synapse`.

### 3. Настройка Synapse

Отредактируйте файл `synapse/homeserver.yaml`:

#### Подключение к базе данных

Найдите секцию `database` и замените на:

```yaml
database:
  name: psycopg2
  args:
    user: synapse
    password: "YOUR_DB_PASSWORD_HERE"  # Ваш пароль из .env
    host: db
    port: 5432
    database: synapse
    cp_min: 5
    cp_max: 10
```

#### Настройка TURN-сервера

Найдите или добавьте секцию `turn_uris`:

```yaml
turn_uris:
  - "turn:your.domain.com:3478?transport=udp"
  - "turn:your.domain.com:3478?transport=tcp"

turn_shared_secret: "YOUR_TURN_SECRET_HERE"  # Ваш TURN секрет

turn_user_lifetime: 86400000
turn_allow_guests: true
```

#### Дополнительные настройки безопасности

```yaml
# Ограничение регистрации
enable_registration: false
enable_registration_without_verification: false

# Настройки медиа
max_upload_size: 50M
max_image_pixels: 32M

# Настройки производительности
event_cache_size: 10K
```

### 4. Конфигурация Coturn

Создайте файл `turnserver.conf`:

```ini
# Основные настройки
listening-port=3478
tls-listening-port=5349

# Диапазон портов для медиа
min-port=49152
max-port=65535

# Аутентификация
use-auth-secret
static-auth-secret=YOUR_TURN_SECRET_HERE

# Домен и IP
realm=your.domain.com
listening-ip=YOUR_SERVER_PUBLIC_IP
relay-ip=YOUR_SERVER_PUBLIC_IP

# Логирование
syslog
verbose

# Безопасность
no-multicast-peers
no-cli
no-tlsv1
no-tlsv1_1
```

### 5. Конфигурация Caddy

Создайте файл `Caddyfile`:

```caddy
your.domain.com {
    # Заголовки безопасности
    header {
        X-Content-Type-Options nosniff
        X-Frame-Options DENY
        X-XSS-Protection "1; mode=block"
        Referrer-Policy strict-origin-when-cross-origin
    }

    # Федерация Matrix
    handle_path /.well-known/matrix/* {
        header Content-Type application/json
        header Access-Control-Allow-Origin *
        header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS"
        header Access-Control-Allow-Headers "Origin, X-Requested-With, Content-Type, Accept, Authorization"
        
        respond /.well-known/matrix/server `{"m.server": "your.domain.com:443"}`
        respond /.well-known/matrix/client `{
            "m.homeserver": {
                "base_url": "https://your.domain.com"
            },
            "m.identity_server": {
                "base_url": "https://vector.im"
            }
        }`
    }

    # Проксирование Matrix API
    reverse_proxy /_matrix/* http://synapse:8008 {
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}
    }
    
    reverse_proxy /_synapse/* http://synapse:8008 {
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}
    }

    # Статические файлы (опционально)
    handle {
        root * /var/www/html
        file_server
    }

    # Логирование
    log {
        output file /var/log/caddy/matrix.log
    }
}
```

---

## Запуск и проверка

### Запуск сервисов

```bash
# Переходим в папку проекта
cd /opt/matrix

# Запускаем все сервисы
sudo docker-compose up -d

# Проверяем статус
sudo docker-compose ps
```

### Проверка логов

```bash
# Общие логи
sudo docker-compose logs

# Логи конкретного сервиса
sudo docker-compose logs synapse
sudo docker-compose logs caddy
sudo docker-compose logs coturn
```

### Проверка федерации

Откройте в браузере:
```
https://federationtester.matrix.org/#your.domain.com
```

---

## Управление регистрацией и пользователями

### Настройка политики регистрации

#### Закрытая регистрация (рекомендуется)

По умолчанию регистрация закрыта. В файле `synapse/homeserver.yaml`:

```yaml
# Регистрация закрыта
enable_registration: false
enable_registration_without_verification: false

# Дополнительная защита
registration_requires_token: true
```

#### Открытая регистрация

**⚠️ ОСТОРОЖНО:** Открытая регистрация может привести к спаму и злоупотреблениям!

```yaml
# Открытая регистрация
enable_registration: true
enable_registration_without_verification: true

# Обязательно добавьте лимиты
registration_shared_secret: "RANDOM_SECRET_STRING"

# Лимиты для предотвращения спама
rc_registration:
  per_second: 0.1  # Максимум 1 регистрация в 10 секунд
  burst_count: 3   # Максимум 3 регистрации подряд

# Требование email для регистрации (рекомендуется)
registrations_require_3pid:
  - email

email:
  smtp_host: smtp.gmail.com
  smtp_port: 587
  smtp_user: your-email@gmail.com
  smtp_pass: your-app-password
  force_tls: true
  notif_from: "Matrix Server <your-email@gmail.com>"
```

#### Регистрация только по токенам

Компромиссный вариант - разрешить регистрацию только по специальным токенам:

```yaml
enable_registration: true
registration_requires_token: true
```

### Создание пользователей

#### Создание администратора

```bash
sudo docker-compose exec synapse register_new_matrix_user \
  -u admin \
  -p STRONG_PASSWORD \
  -a \
  -c /data/homeserver.yaml \
  http://localhost:8008
```

#### Создание обычного пользователя (закрытая регистрация)

```bash
sudo docker-compose exec synapse register_new_matrix_user \
  -u username \
  -p PASSWORD \
  -c /data/homeserver.yaml \
  http://localhost:8008
```

#### Создание токенов для регистрации

**Современный способ через Admin API:**

```bash
# Получите admin access token через Element клиент:
# Settings → Help & About → Advanced → Access Token

# Создание одноразового токена
curl -X POST \
  "https://your.domain.com/_synapse/admin/v1/registration_tokens" \
  -H "Authorization: Bearer YOUR_ADMIN_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "uses_allowed": 1,
    "length": 16
  }'

# Создание токена с ограничениями по времени и использованию
curl -X POST \
  "https://your.domain.com/_synapse/admin/v1/registration_tokens" \
  -H "Authorization: Bearer YOUR_ADMIN_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "uses_allowed": 5,
    "expiry_time": 1735689599000,
    "length": 20
  }'

# Список всех токенов
curl -X GET \
  "https://your.domain.com/_synapse/admin/v1/registration_tokens" \
  -H "Authorization: Bearer YOUR_ADMIN_ACCESS_TOKEN"

# Удаление токена
curl -X DELETE \
  "https://your.domain.com/_synapse/admin/v1/registration_tokens/TOKEN_HERE" \
  -H "Authorization: Bearer YOUR_ADMIN_ACCESS_TOKEN"
```

**Альтернативный способ (устаревший):**

```bash
# Для старых версий Synapse
sudo docker-compose exec synapse \
  register_new_matrix_user \
  -c /data/homeserver.yaml \
  http://localhost:8008
```

### Управление пользователями через Admin API

#### Активация Admin API

Добавьте в `homeserver.yaml`:

```yaml
# Включаем Admin API
enable_registration: false
admin_contact: 'mailto:admin@your.domain.com'

# Настройки для администраторов
server_notices:
  system_mxid_localpart: notices
  system_mxid_display_name: "Server Notices"
  room_name: "Server Notices"
```

#### Полезные команды Admin API

**Получение admin access token:**
1. Войдите в Element как администратор
2. Settings → Help & About → Advanced → Access Token
3. Скопируйте токен (начинается с `syt_`)

```bash
# Переменная для удобства
ADMIN_TOKEN="YOUR_ADMIN_ACCESS_TOKEN_HERE"
BASE_URL="https://your.domain.com"

# Список всех пользователей
curl -X GET \
  "$BASE_URL/_synapse/admin/v2/users" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Информация о конкретном пользователе
curl -X GET \
  "$BASE_URL/_synapse/admin/v2/users/@username:your.domain.com" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Деактивация пользователя
curl -X POST \
  "$BASE_URL/_synapse/admin/v1/deactivate/@username:your.domain.com" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"erase": false}'

# Получение статистики сервера
curl -X GET \
  "$BASE_URL/_synapse/admin/v1/statistics/users/media" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Список комнат пользователя
curl -X GET \
  "$BASE_URL/_synapse/admin/v1/users/@username:your.domain.com/joined_rooms" \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# Принудительное обновление статистики комнаты
curl -X POST \
  "$BASE_URL/_synapse/admin/v1/rooms/!roomid:your.domain.com/forward_extremities" \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

### Переключение режимов регистрации

#### Скрипт для быстрого переключения

Создайте файл `toggle_registration.sh`:

```bash
#!/bin/bash

CONFIG_FILE="/opt/matrix/synapse/homeserver.yaml"
COMPOSE_DIR="/opt/matrix"

echo "Текущее состояние регистрации:"
grep "enable_registration:" $CONFIG_FILE

echo ""
echo "1) Закрыть регистрацию"
echo "2) Открыть регистрацию"
echo "3) Регистрация только по токенам"
read -p "Выберите опцию (1-3): " choice

case $choice in
  1)
    sed -i 's/enable_registration: true/enable_registration: false/' $CONFIG_FILE
    sed -i 's/registration_requires_token: false/registration_requires_token: true/' $CONFIG_FILE
    echo "Регистрация закрыта"
    ;;
  2)
    sed -i 's/enable_registration: false/enable_registration: true/' $CONFIG_FILE
    sed -i 's/registration_requires_token: true/registration_requires_token: false/' $CONFIG_FILE
    echo "⚠️ ВНИМАНИЕ: Регистрация открыта для всех!"
    ;;
  3)
    sed -i 's/enable_registration: false/enable_registration: true/' $CONFIG_FILE
    sed -i 's/registration_requires_token: false/registration_requires_token: true/' $CONFIG_FILE
    echo "Регистрация доступна только по токенам"
    ;;
  *)
    echo "Неверный выбор"
    exit 1
    ;;
esac

echo "Перезапускаем Synapse..."
cd $COMPOSE_DIR
sudo docker-compose restart synapse

echo "Готово! Новые настройки применены."
```

Сделайте скрипт исполняемым:

```bash
chmod +x toggle_registration.sh
```

### Мониторинг регистраций

#### Просмотр новых регистраций

```bash
# Последние регистрации
sudo docker-compose exec db psql -U synapse -c "
  SELECT name, creation_ts, admin, deactivated 
  FROM users 
  WHERE user_type IS NULL 
  ORDER BY creation_ts DESC 
  LIMIT 10;"

# Статистика по дням
sudo docker-compose exec db psql -U synapse -c "
  SELECT 
    DATE(to_timestamp(creation_ts/1000)) as date,
    COUNT(*) as registrations
  FROM users 
  WHERE user_type IS NULL 
  GROUP BY DATE(to_timestamp(creation_ts/1000))
  ORDER BY date DESC 
  LIMIT 30;"
```

#### Настройка уведомлений о регистрациях

Добавьте в `homeserver.yaml`:

```yaml
# Уведомления администратору
server_notices:
  system_mxid_localpart: notices
  system_mxid_display_name: "Server Notices"
  room_name: "Server Notices"

# Логирование регистраций
loggers:
  synapse.handlers.register:
    level: INFO
```

### Защита от спама

#### Настройки rate limiting

```yaml
# Лимиты для регистрации
rc_registration:
  per_second: 0.17  # ~1 регистрация в 6 секунд
  burst_count: 3

# Лимиты для сообщений
rc_message:
  per_second: 0.2
  burst_count: 10

# Лимиты для входа
rc_login:
  address:
    per_second: 0.17
    burst_count: 3
  account:
    per_second: 0.17
    burst_count: 3
  failed_attempts:
    per_second: 0.17
    burst_count: 3

# Дополнительная защита
require_auth_for_profile_requests: true
limit_profile_requests_to_users_who_share_rooms: true
```

#### Blacklist доменов

```yaml
# Блокировка проблемных доменов
federation_blacklist:
  - "bad-server.com"
  - "spam-server.org"

# Список разрешенных доменов (whitelist)
# federation_whitelist:
#   - "matrix.org"
#   - "trusted-server.com"
```

---

## Диагностика и решение проблем

### Частые проблемы

#### Проблема: Контейнер не запускается

```bash
# Проверяем логи
sudo docker-compose logs SERVICE_NAME

# Проверяем конфигурацию
sudo docker-compose config
```

#### Проблема: Нет SSL-сертификата

```bash
# Проверяем логи Caddy
sudo docker-compose logs caddy

# Проверяем DNS
nslookup your.domain.com
```

#### Проблема: Не работают звонки

```bash
# Проверяем Coturn
sudo docker-compose logs coturn

# Проверяем порты
sudo netstat -tulpn | grep 3478
```

### Полезные команды

```bash
# Перезапуск сервиса
sudo docker-compose restart synapse

# Обновление образов
sudo docker-compose pull
sudo docker-compose up -d

# Очистка логов
sudo docker system prune -f
```

---

## Обслуживание

### Резервное копирование

```bash
#!/bin/bash
# Скрипт backup.sh

BACKUP_DIR="/opt/backups/matrix"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Останавливаем сервисы
cd /opt/matrix
sudo docker-compose stop

# Создаем архив
sudo tar -czf $BACKUP_DIR/matrix_backup_$DATE.tar.gz \
  /opt/matrix/synapse \
  /opt/matrix/postgres \
  /opt/matrix/docker-compose.yml \
  /opt/matrix/Caddyfile \
  /opt/matrix/turnserver.conf

# Запускаем сервисы
sudo docker-compose start

# Удаляем старые бэкапы (старше 30 дней)
find $BACKUP_DIR -name "matrix_backup_*.tar.gz" -mtime +30 -delete
```

### Обновление

```bash
# Создаем бэкап перед обновлением
./backup.sh

# Обновляем образы
cd /opt/matrix
sudo docker-compose pull

# Перезапускаем с новыми образами
sudo docker-compose up -d
```

### Мониторинг

```bash
# Использование ресурсов
sudo docker stats

# Размер базы данных
sudo docker-compose exec db psql -U synapse -c "
  SELECT pg_size_pretty(pg_database_size('synapse'));"

# Количество пользователей
sudo docker-compose exec db psql -U synapse -c "
  SELECT COUNT(*) FROM users WHERE user_type IS NULL;"
```

---

## Заключение

Поздравляем! Вы успешно развернули собственный Matrix-сервер. Теперь вы можете:

1. **Подключиться через Element:** https://app.element.io
2. **Использовать собственный домен** при входе
3. **Создавать комнаты** и приглашать пользователей
4. **Общаться с федерацией** других Matrix-серверов

### Полезные ссылки

- [Официальная документация Matrix](https://matrix.org/docs/)
- [Synapse документация (новый репозиторий Element)](https://element-hq.github.io/synapse/latest/)
- [Synapse документация (Matrix.org - до версии 1.99)](https://matrix-org.github.io/synapse/latest/)
- [Synapse Admin API](https://element-hq.github.io/synapse/latest/usage/administration/admin_api/)
- [Element клиент](https://element.io/)
- [Список публичных серверов](https://joinmatrix.org/servers/)
- [Matrix Specification](https://spec.matrix.org/)
- [Coturn документация](https://github.com/coturn/coturn)

