# Миграция на VPN-сервис — существующий VPS

Твой текущий сервер: 3x-ui, Mimo Code, домены, поддомены, Telegram-боты уже есть.
Нужно: почистить, поставить Docker, поднять этот проект, перенастроить DNS/ботов.

---

## Шаг 0. Проверки перед началом 

```bash
ssh root@<IP>
uname -a          # Ubuntu 20.04+ или Debian 11+
free -h           # минимум 2 ГБ RAM (лучше 4 ГБ)
df -h /           # минимум 15 ГБ свободного места
✅
```

Если RAM < 2 ГБ — сервер не потянет. Минимум: 2 vCPU + 2 ГБ RAM + 2 ГБ swap.
---
✅

## Шаг 1. Очистка сервера

### 1.1. Остановить и удалить 3x-ui

```bash
# Поднимись: какой порт у 3x-ui?
# Обычно 2096 или другой. Посмотри:
ss -tlnp | grep 3x
docker ps         # если 3x-ui в docker

# Если в docker:
docker compose -f /opt/3x-ui/docker-compose.yml down 2>/dev/null || \
docker rm -f 3x-ui 2>/dev/null || true

# Удалить файлы 3x-ui
rm -rf /opt/3x-ui /usr/local/bin/xui /etc/x-ui 2>/dev/null || true

# Если был системный сервис:
systemctl disable x-ui 2>/dev/null || true
systemctl stop x-ui 2>/dev/null || true
rm -f /etc/systemd/system/x-ui.service 2>/dev/null || true
systemctl daemon-reload
```
✅

### 1.2. Удалить Mimo Code

```bash
# Где обычно стоит? Проверь:
which mimo-code 2>/dev/null || which mimocode 2>/dev/null || true
ls /opt/mimo* 2>/dev/null || true
ls /usr/local/bin/mimo* 2>/dev/null || true

# Если в docker:
docker ps | grep mimo
docker compose -f /opt/mimo-code/docker-compose.yml down 2>/dev/null || \
docker rm -f mimo-code 2>/dev/null || true

# Удалить файлы
rm -rf /opt/mimo* /usr/local/bin/mimo* 2>/dev/null || true
systemctl disable mimo-code 2>/dev/null || true
systemctl stop mimo-code 2>/dev/null || true
rm -f /etc/systemd/system/mimo*.service 2>/dev/null || true
systemctl daemon-reload
```
✅

### 1.3. Удалить лишние доккер-контейнеры и образы

```bash
# Посмотреть всё, что запущено
docker ps -a

# Остановить и удалить ВСЁ
docker compose -f /opt/*/docker-compose.yml down 2>/dev/null || true
docker rm -f $(docker ps -aq) 2>/dev/null || true
docker rmi --force $(docker images -aq) 2>/dev/null || true
docker volume prune -f
docker network prune -f
```
✅

### 1.4. Остановить ненужные сервисы

```bash
# Посмотреть все сервисы
systemctl list-units --type=service --state=running

# Остановить то, что не нужно (SSH, firewall — оставить!)
# Примеры: nginx, apache, node-утилиты от старого кода
# Каждый раз проверяй, не убьёшь ли себе SSH

# Удалить старые репозитории npm/yarn если есть
npm uninstall -g node-red 2>/dev/null || true
npm uninstall -g pm2 2>/dev/null || true
```
✅

### 1.5. Почистить пользователей

```bash
# Посмотреть всех пользователей
cat /etc/passwd | grep -v nologin | grep -v false

# Должны остаться: root и camarik
# Удалить остальных (заменить <user> на реальные имена):
# userdel -r <user>
# groupdel <group>

# Проверить cron у чужих пользователей
# crontab -u <user> -l 2>/dev/null

# Очистить/tmp
rm -rf /tmp/*
```
✅

### 1.6. Очистка портов

```bash
# Посмотреть, что слушает
ss -tlnp

# Должны остаться только:
#   22/tcp — SSH
#   80/tcp — будет Caddy
#   443/tcp — будет Caddy
#   443/udp — будет HTTP/3
#   8443/tcp — будет VLESS REALITY
# Всё остальное — убить.
```
✅

### 1.7. Фаервол — чистый

```bash
ufw status
# Если UFW не настроен — хорошо.
# Если настроен — сбросить:
ufw reset
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443
ufw allow 443/udp
ufw allow 8443/tcp
ufw enable
```
✅

---

## Шаг 2. Установить Docker

```bash
curl -fsSL https://get.docker.com | sh
usermod -aG docker camarik
newgrp docker

docker compose version   # должно быть v2.x
```

---

## Шаг 3. Настроить SSH для camarik

```bash
# Если пользователя camarik ещё нет:
adduser camarik

# Дать sudo
usermod -aG sudo camarik   # Debian
# или
usermod -aG wheel camarik  # Ubuntu

# Настроить SSH-ключ (от LOCAL машины):
mkdir -p /home/camarik/.ssh
cp ~/.ssh/id_ed25519.pub /home/camarik/.ssh/authorized_keys
chown -R camarik:camarik /home/camarik/.ssh
chmod 700 /home/camarik/.ssh
chmod 600 /home/camarik/.ssh/authorized_keys

# Проверить: ssh camarik@<IP> должно работать по ключу
```

**Важно:** не закрывай SSH сессии до проверки!

---

## Шаг 4. Проверить DNS

Ты используешь существующие домены. Проверь, что они смотрят на IP сервера:

```bash
# Сделай с локальной машины:
dig +short app.<твой_домен>
dig +short sub.<твой_домен>
dig +short <твой_домен>   # если есть apex-домен

# Обе должны вернуть IP твоего VPS.
# Если нет — поправь A-записи в панели регистратора/DNS.
```

Минимальная таблица записей:

| Type | Name | Data |
|---|---|---|
| A | `app` | IP сервера |
| A | `sub` | IP сервера |
| A | (пусто или `@`) | IP сервера |

---

## Шаг 5. Скачать проект на сервер

```bash
ssh root@<IP>
cd /opt

git clone https://github.com/<your-username>/vpn-service.git
cd vpn-service

# Или архивом, если git не нужен:
# scp -r . root@<IP>:/opt/vpn-service/
```

---

## Шаг 6. Заполнить .env

```bash
./scripts/init-env.sh
```

Затем отредактируй `.env` и заполни переменные. Вот таблица того, что нужно:

### Домены (возьми существующие)

| Переменная | Пример | Откуда взять |
|---|---|---|
| `APP_DOMAIN` | `app.mydomain.com` | существующий поддомен |
| `SUB_DOMAIN` | `sub.mydomain.com` | существующий поддомен |
| `ACME_EMAIL` | `admin@mydomain.com` | любая живая почта |
| `PUBLIC_APP_URL` | `https://app.mydomain.com` | `https://APP_DOMAIN` |
| `MARZBAN_SUB_URL_PREFIX` | `https://sub.mydomain.com` | `https://SUB_DOMAIN` |

> `APP_DOMAIN` и `SUB_DOMAIN` — четыре места в `.env`, все должны совпадать.

### Telegram (возьми токены существующих ботов)

| Переменная | Где взять |
|---|---|
| `TELEGRAM_BOT_TOKEN` | [@BotFather](https://t.me/BotFather) — `/mybots`, выбери бота, API Token |
| `TELEGRAM_BOT_USERNAME` | username бота без `@` |
| `TELEGRAM_WEBHOOK_SECRET` | `openssl rand -hex 16` — случайная строка |
| `ADMIN_CHAT_ID` | [@userinfobot](https://t.me/userinfobot) — отправь `/start`, получишь свой ID |

### Telegram Mini App

| Переменная | Где взять |
|---|---|
| `RETURN_DEEPLINK` | `https://t.me/<BOT_USERNAME>/<app_short_name>` — `<app_short_name>` от BotFather при создании Mini App (`/newapp`) |

### Marzban (админские аккаунты панели)

| Переменная | Что делать |
|---|---|
| `MARZBAN_ADMIN_USERNAME` | Оставить `vpnapp` — это аккаунт приложения к панели |
| `MARZBAN_ADMIN_PASSWORD` | Сгенерирован автоматически `init-env.sh` |
| `MARZBAN_SUDO_USERNAME` | Опционально — оставь пустым, если не нужен дашборд оператора |
| `MARZBAN_SUDO_PASSWORD` | Сгенерирован автоматически `init-env.sh` |

### Маршруты REALITY (оставь по умолчанию)

| Переменная | Значение |
|---|---|
| `REALITY_PORT` | `8443` |
| `REALITY_DEST` | `gateway.icloud.com:443` |
| `REALITY_SERVER_NAMES` | `gateway.icloud.com` |
| `REALITY_PRIVATE_KEY` | **пусто** — сгенерируется при первом запуске |
| `REALITY_SHORT_ID` | **пусто** — сгенерируется при первом запуске |

### Платежи

| Переменная | Что делать |
|---|---|
| `PAYMENT_PROVIDER` | `stripe` или `fake` (для теста) |
| `STRIPE_SECRET_KEY` | `sk_test_...` из дашборда Stripe |
| `STRIPE_WEBHOOK_SECRET` | `whsec_...` после настройки вебхука (шаг 9) |
| `PRICE_CURRENCY` | `usd` |

### Не трогать

| Переменная | Почему |
|---|---|
| `NODE_ENV` | `production` |
| `DATABASE_PATH` | `/data/app.db` |
| `SESSION_TTL_DAYS` | `7` |
| `MARZBAN_API_URL` | `http://marzban:8000` |
| `MARZBAN_INBOUND_TAGS` | `VLESS TCP REALITY` |
| `MARZBAN_VLESS_FLOW` | `xtls-rprx-vision` |

---

## Шаг 7. Запустить

```bash
./scripts/deploy.sh
```

Первый запуск: 5–15 минут. Скрипт:
1. Соберёт образ приложения (`npm run build` внутри Docker)
2. Запустит миграции БД (`app-migrate`)
3. Настроит Marzban: REALITY-ключи + миграции панели + админы (`marzban-init`)
4. Поднимет панель Marzban
5. Запустит приложение
6. Проверит, что приложение подключается к панели (`marzban-check`)

### Успех выглядит так:

```
deploy:   app-migrate: ok
deploy:   marzban-init: ok
deploy:   marzban-check: ok
deploy: done — the app, the panel and the app-to-panel path all check out
```

---

## Шаг 8. Настроить Telegram-бота

Только **после** того, как Caddy получил TLS-сертификат (проверь `curl -I https://app.<домен>` → `HTTP/2 200`):

```bash
# 1. Зарегистрировать Mini App у BotFather:
#    /newapp → имя → описание → бот → URL (https://app.<домен>)
#    Запомни short name.

# 2. Установить вебхук:
docker compose --profile webhook up app-set-webhook

# Должно быть: lastError=none
```

### Обновить RETURN_DEEPLINK

Если short name от BotFather отличается от текущего `RETURN_DEEPLINK` в `.env`:

```bash
# 1. Поправить в .env
# 2. Пересобрать:
./scripts/deploy.sh
```

---

## Шаг 9. Настроить Stripe (или оставить fake)

Если `PAYMENT_PROVIDER=fake` — всё работает без Stripe. Для реальных платежей:

1. Зайти в [Stripe Dashboard](https://dashboard.stripe.com)
2. Webhooks → Add endpoint:
   - URL: `https://app.<твой_домен>/api/stripe/webhook`
   - Events: `checkout.session.completed`, `checkout.session.async_payment_succeeded`, `checkout.session.async_payment_failed`, `checkout.session.expired`
3. Скопировать `whsec_...` → в `.env` как `STRIPE_WEBHOOK_SECRET`
4. `./scripts/deploy.sh`

---

## Шаг 10. Финальная проверка

```bash
docker compose ps -a
```

`caddy`, `app`, `marzban` → `Up`. Три one-shot → `Exited (0)`.

Ручные проверки:
1. Открыть `https://app.<домен>` из Telegram — мини-апп загружается
2. `/start` в боте → ссылка на мини-апп
3. Купить тариф (Stripe test или fake)
4. В профиле — QR-код и ссылка подписки
5. Импортировать ссылку в v2rayNG → подключение работает

---

## После деплоя

### Доступ к дашборду Marzban

Через SSH-туннель:
```bash
ssh -L 8000:127.0.0.1:8000 camarik@<IP>
# Открыть http://localhost:8000/dashboard/
```

### Обновить код

```bash
cd /opt/vpn-service
git pull
./scripts/deploy.sh
```

### Бэкапы

```bash
apt install -y sqlite3
cp scripts/backup.sh /usr/local/bin/vpn-backup
chmod 700 /usr/local/bin/vpn-backup
crontab -e    # добавить строку из scripts/backup.cron.example
```

---

## Чего НЕ делать

- `docker compose down -v` — удалит volume с ключами REALITY и БД
- Не открывать порт 8000 наружу — это панель Marzban
- Не ставить `REALITY_DEST` на свой домен — выдаст вас через SNI
