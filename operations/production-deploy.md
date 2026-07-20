# MarketHacker — production-деплой на чистый VPS (Ubuntu 24.04)

Полное руководство по развёртыванию всего стека с нуля на одном VPS (в т.ч. **VK Cloud**).  
Основной сценарий: **ручной деплой по SSH** от пользователя `ubuntu` (стандартный аккаунт образа).  
Инструкция опирается на фактические compose/Makefile/Dockerfile репозиториев.

**Каноническая структура на сервере:**

```text
/opt/markethacker/
  infra/              # Docker-сети + backup-скрипты
  backend/            # API + worker + PG/Redis/PgBouncer
  parser/             # API + worker + PG/Redis/PgBouncer + ClickHouse + Kafka
  admin-panel/        # Next.js → :3001
  manager-portal/     # Next.js → :3002
  market-navigators/  # лендинг markethacker.ru → :3003
  caddy/              # reverse proxy + TLS + статика api.markethacker.ru/
  monitoring/         # Prometheus + Grafana + exporters
  backups/            # архивы бэкапов (права 700)
```

---

## 0. Что поднимается

| Стек | Контейнеры | Порт на хосте |
|------|------------|---------------|
| backend infra | postgres, pgbouncer, redis | только Docker-сеть |
| backend app | api, worker | `127.0.0.1:8000` |
| parser infra | postgres, pgbouncer, redis, clickhouse, kafka | только Docker-сеть (+ clickhouse alias в `markethacker_apps`) |
| parser app | api (`markethacker-parser-api`), worker | `127.0.0.1:8010` |
| admin-panel | admin-panel | `127.0.0.1:3001` |
| manager-portal | manager-portal | `127.0.0.1:3002` |
| market-navigators | market-navigators | `127.0.0.1:3003` |
| caddy | caddy (host network) | `:80`, `:443` |
| monitoring | prometheus, grafana, alertmanager, node-exporter, cadvisor, blackbox, pg/redis/kafka exporters | `127.0.0.1:9090/3000/9093/9100/8081/9115` + exporters `9187–9308` |
| parser ClickHouse metrics | (в parser infra) | `127.0.0.1:9363` |
| backend/parser workers metrics | (в prod compose) | `127.0.0.1:9101` / `9102` |

**Не деплоится на VPS:** `docs`, `extension-chrome` (Chrome Web Store).

**Очереди:** ARQ поверх Redis (два независимых Redis). **Kafka** — буфер parser → ClickHouse.

**Сертификаты:** Let's Encrypt через Caddy (volume `caddy_data`). Email ACME: `admin@markethacker.ru`.

---

## 1. Требования к VPS

### ОС

- **Ubuntu 24.04 LTS** (x86_64). На VK Cloud по умолчанию пользователь **`ubuntu`** — им и пользуемся для всего: настройки сервера, `git`, Docker, обновления.

### Ресурсы (ориентир для текущего single-node стека)

На одном хосте живут 2×PostgreSQL, 2×Redis, ClickHouse (~30% RAM), Kafka (~0.5–1 GB heap), 4 app-контейнера, Caddy, мониторинг.

| Профиль | vCPU | RAM | Диск |
|---------|------|-----|------|
| Минимум (риск OOM при пиках парсинга) | 4 | 16 GB | 160 GB SSD |
| Рекомендуемый | 8 | 32 GB | 300+ GB NVMe/SSD |
| Комфортный | 8–16 | 64 GB | 500 GB |

Проверьте фактические ресурсы:

```bash
nproc
free -h
df -h /
lsblk
```

Если RAM < 16 GB — перед продом уменьшите `KAFKA_HEAP_OPTS` (например `-Xmx384m -Xms192m`) и следите за ClickHouse (`max_server_memory_usage_to_ram_ratio=0.3` уже в конфиге parser).

### Диск и ФС

- Раздел `/` на **ext4** или **xfs** (типично для Ubuntu cloud-image).
- Отдельный диск под Docker volumes желателен, но не обязателен на старте.
- Не ставьте Docker root на сетевой NFS.

### Swap

На машине с 32 GB:

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
sudo sysctl vm.swappiness=10
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swapiness.conf
```

Swap — страховка от кратковременных пиков, не замена RAM.

### Timezone / locale / NTP

```bash
sudo timedatectl set-timezone Europe/Moscow
sudo timedatectl set-ntp true
sudo apt-get update
sudo apt-get install -y locales
sudo locale-gen en_US.UTF-8 ru_RU.UTF-8
sudo update-locale LANG=en_US.UTF-8
timedatectl status
```

### Обновление системы

```bash
sudo apt-get update
sudo apt-get -y upgrade
sudo apt-get -y autoremove
sudo reboot   # если обновилось ядро
```

### Firewall (UFW)

Открыты только SSH, HTTP, HTTPS. Все app-порты слушают `127.0.0.1`.

```bash
sudo apt-get install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
sudo ufw status verbose
```

Если SSH на нестандартном порту — разрешите его **до** `enable`.

---

## 2. Пользователь `ubuntu` и каталог `/opt`

Отдельный системный пользователь для деплоя **не создаём** — достаточно `ubuntu` (уже есть sudo).

`Permission denied` при `git clone /opt/...` бывает потому, что `/opt` принадлежит root. Решение — один раз отдать каталог проекта пользователю `ubuntu`:

```bash
sudo apt-get install -y git curl ca-certificates gnupg fail2ban unattended-upgrades

sudo mkdir -p /opt/markethacker/backups
sudo chown -R ubuntu:ubuntu /opt/markethacker
sudo chmod 750 /opt/markethacker
sudo chmod 700 /opt/markethacker/backups
```

После этого все команды в инструкции выполняйте от **`ubuntu`** (без `sudo`, кроме установки пакетов и правок `/etc`).

Процессы приложений работают внутри Docker (не от root хоста): Python — uid `10001`, Next.js — `nextjs`, landing — `app`.

---

## 3. SSH

На **рабочей машине** (если ещё не заходите по ключу):

```bash
ssh-keygen -t ed25519 -f ~/.ssh/markethacker_vps -C "markethacker-vps"
ssh-copy-id -i ~/.ssh/markethacker_vps.pub ubuntu@<VPS_IP>
ssh -i ~/.ssh/markethacker_vps ubuntu@<VPS_IP>
```

### Hardening SSH (рекомендуется)

```bash
sudo tee /etc/ssh/sshd_config.d/99-markethacker.conf >/dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AllowUsers ubuntu
X11Forwarding no
MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 120
ClientAliveCountMax 2
EOF

sudo sshd -t
sudo systemctl reload ssh
```

Перед `reload` убедитесь, что вход по ключу как `ubuntu` уже работает — иначе можно потерять доступ.

### fail2ban

```bash
sudo systemctl enable --now fail2ban
sudo tee /etc/fail2ban/jail.d/sshd.local >/dev/null <<'EOF'
[sshd]
enabled = true
port = ssh
filter = sshd
maxretry = 5
bantime = 1h
findtime = 10m
EOF
sudo systemctl restart fail2ban
```

---

## 4. Docker

От **`ubuntu`**:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker ubuntu
```

Выйдите из SSH и зайдите снова (чтобы применилась группа `docker`), затем:

```bash
docker version
docker compose version
```

### daemon.json (log rotation)

```bash
sudo tee /etc/docker/daemon.json >/dev/null <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  },
  "live-restore": true,
  "userland-proxy": false,
  "no-new-privileges": true
}
EOF
sudo systemctl restart docker
```

- **storage driver:** `overlay2` (дефолт Docker CE на Ubuntu 24.04).
- **restart policy:** в prod-compose уже `unless-stopped`.

---

## 5. Доступ к GitHub (приватные репозитории)

Самый простой способ для ручного деплоя: **один SSH-ключ пользователя `ubuntu`**, добавленный в **ваш личный** GitHub-аккаунт (Settings → SSH and GPG keys).  
Этот ключ даёт доступ ко всем репозиториям, куда у вас есть права — отдельно на каждый repo ничего вешать не нужно.

> Deploy keys GitHub **не используем**: один ключ нельзя повесить на несколько репозиториев.

На сервере от `ubuntu`:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "ubuntu@markethacker-vps"
cat ~/.ssh/id_ed25519.pub
# Скопируйте вывод → GitHub → ваш аккаунт → Settings → SSH and GPG keys → New SSH key

ssh -T git@github.com
# Hi <ваш-логин>! You've successfully authenticated...
```

Дальше обычный clone:

```bash
git clone git@github.com:MarketHackerLabs/backend.git
```

Если репозитории публичные — ключ для clone не обязателен.

Альтернатива без SSH: HTTPS + Personal Access Token (Contents: Read) при `git clone` / `git pull`. Токен не вставляйте в URL в истории команд — лучше ввести по запросу или через `gh auth login`.

---

## 6. DNS (до поднятия Caddy)

В DNS-зоне `markethacker.ru` создайте A/AAAA на IP VPS:

| Имя | Тип |
|-----|-----|
| `markethacker.ru` | A |
| `www.markethacker.ru` | A (или CNAME → apex) |
| `api.markethacker.ru` | A |
| `admin.markethacker.ru` | A |
| `team.markethacker.ru` | A |
| `wb-proxy.markethacker.ru` | A |
| `wb-connect.markethacker.ru` | A |
| все onboarding-хосты из `caddy/Caddyfile` (seller, cmp, …) **или** wildcard `*.wb-connect.markethacker.ru` | A |
| `grafana.markethacker.ru` | A |

Проверка: `dig +short api.markethacker.ru`.

Caddy не выпустит сертификаты, пока DNS не указывает на этот сервер (порты 80/443 открыты).

---

## 7. Клонирование репозиториев

От **`ubuntu`** (каталог уже принадлежит вам):

```bash
cd /opt/markethacker

git clone git@github.com:MarketHackerLabs/infra.git
git clone git@github.com:MarketHackerLabs/backend.git
git clone git@github.com:MarketHackerLabs/parser.git
git clone git@github.com:MarketHackerLabs/admin-panel.git
git clone git@github.com:MarketHackerLabs/manager-portal.git
git clone git@github.com:MarketHackerLabs/caddy.git
git clone git@github.com:MarketHackerLabs/monitoring.git
git clone git@github.com:DrRobotGranata/market-navigators.git   # или актуальный org remote
```

> Makefiles backend/parser ожидают `../infra` рядом — структура `/opt/markethacker/{infra,backend,parser}` это обеспечивает.

Обновление позже: в каталоге сервиса `git pull` (или `git fetch && git checkout <tag>`), затем `make prod-migrate` / `make prod-up`. Файл `.env` в git не попадает — его не затирайте.
---

## 8. Переменные окружения

### Где хранить

| Файл | Владелец | Права |
|------|----------|-------|
| `/opt/markethacker/<service>/.env` | `ubuntu:ubuntu` | `600` |
| `/opt/markethacker/backups/*.tar.gz` | `ubuntu:ubuntu` | `600` |
| `/opt/markethacker/backups/` | `ubuntu:ubuntu` | `700` |

**Нельзя в Git:** любые `.env`, приватные ключи, дампы БД, `caddy_data` с сертификатами (LE перевыпустит, но бэкап полезен).

Резервное копирование `.env` входит в `infra/scripts/backup-all.sh`.

### Генерация секретов

```bash
openssl rand -hex 32   # JWT_SECRET, ENCRYPTION_KEY, PARSER_API_KEY, REDIS_PASSWORD, …
openssl rand -base64 24  # Grafana password
```

### Backend `.env` (production)

```bash
cd /opt/markethacker/backend
cp .env.example .env
chmod 600 .env
```

Обязательно выставить (значения — пример структуры):

```bash
ENVIRONMENT=production
LOG_LEVEL=INFO

POSTGRES_USER=markethacker
POSTGRES_PASSWORD=<strong>
POSTGRES_DB=markethacker

REDIS_PASSWORD=<strong>
DATABASE_URL=postgresql+asyncpg://markethacker:<strong>@pgbouncer:6432/markethacker
DATABASE_DIRECT_URL=postgresql+asyncpg://markethacker:<strong>@postgres:5432/markethacker
DATABASE_USE_PGBOUNCER=true
REDIS_URL=redis://:<strong>@redis:6379/0

JWT_SECRET=<openssl rand -hex 32>
ENCRYPTION_KEY=<openssl rand -hex 32>
ENCRYPTION_KEY_VERSION=1

PARSER_API_KEY=<тот же, что в parser/.env>
# PARSER_SERVICE_URL / CLICKHOUSE_* переопределяются в docker-compose.prod.yml для apps-сети,
# но CLICKHOUSE_PASSWORD/USER/DATABASE должны совпадать с parser:
CLICKHOUSE_ENABLED=true
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=<как в parser>
CLICKHOUSE_DATABASE=markethacker_parser

CORS_ORIGINS=["https://team.markethacker.ru","https://admin.markethacker.ru","chrome-extension://<id>"]

WB_PORTAL_PUBLIC_BASE_URL=https://wb-proxy.markethacker.ru
WB_CONNECT_PUBLIC_BASE_URL=https://wb-connect.markethacker.ru/api/v1/wb-connect
WB_PORTAL_COOKIE_SECURE=true

# Платежи
YOOKASSA_SHOP_ID=...
YOOKASSA_SECRET_KEY=...
YOOKASSA_DEFAULT_RECEIPT_EMAIL=admin@markethacker.ru
STRIPE_SECRET_KEY=...
STRIPE_WEBHOOK_SECRET=...

SENTRY_DSN=   # опционально
```

### Parser `.env`

```bash
cd /opt/markethacker/parser
cp .env.example .env
chmod 600 .env
```

```bash
ENVIRONMENT=production
LOG_LEVEL=INFO

POSTGRES_USER=markethacker_parser
POSTGRES_PASSWORD=<strong>
POSTGRES_DB=markethacker_parser

REDIS_PASSWORD=<strong>
DATABASE_URL=postgresql+asyncpg://markethacker_parser:<strong>@pgbouncer:6432/markethacker_parser
DATABASE_DIRECT_URL=postgresql+asyncpg://markethacker_parser:<strong>@postgres:5432/markethacker_parser
DATABASE_USE_PGBOUNCER=true
REDIS_URL=redis://:<strong>@redis:6379/0

PARSER_API_KEY=<совпадает с backend>

CLICKHOUSE_ENABLED=true
CLICKHOUSE_HOST=clickhouse
CLICKHOUSE_PORT=8123
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=<strong>
CLICKHOUSE_DATABASE=markethacker_parser

KAFKA_ENABLED=true
KAFKA_BOOTSTRAP_SERVERS=kafka:9092
# при малой RAM:
# KAFKA_HEAP_OPTS=-Xmx384m -Xms192m

CORS_ORIGINS=["https://admin.markethacker.ru"]
```

### Frontend `.env`

```bash
# admin-panel
cd /opt/markethacker/admin-panel
cp .env.example .env
# NEXT_PUBLIC_API_URL=https://api.markethacker.ru/api/v1

# manager-portal
cd /opt/markethacker/manager-portal
cp .env.example .env
# NEXT_PUBLIC_API_URL=https://api.markethacker.ru/api/v1
```

`NEXT_PUBLIC_*` вшиваются на **build** — после смены URL нужна пересборка образа.

### Caddy `.env`

```bash
cd /opt/markethacker/caddy
cp .env.example .env
# make hash-password PASS='...'
METRICS_USER=prometheus
METRICS_HASH=<bcrypt>
```

### Monitoring `.env`

```bash
cd /opt/markethacker/monitoring
cp .env.example .env
# Обязательно:
GRAFANA_ADMIN_PASSWORD=<strong>
BACKEND_PG_DSN=postgresql://…@postgres:5432/markethacker?sslmode=disable
PARSER_PG_DSN=postgresql://…@postgres:5432/markethacker_parser?sslmode=disable
BACKEND_REDIS_PASSWORD=…
PARSER_REDIS_PASSWORD=…
# Telegram (Alertmanager). РФ: см. TELEGRAM_API_URL / ALERTMANAGER_HTTP_PROXY в .env.example
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=
```

---

## 9. Порядок первого запуска

Все шаги от **`ubuntu`**, каталог `/opt/markethacker`.

### 9.1. Docker-сети

```bash
cd /opt/markethacker/infra
make networks
make status
```

### 9.2. Backend infrastructure + app

```bash
cd /opt/markethacker/backend
make infra-up-prod          # postgres, pgbouncer, redis
make prod-migrate           # alembic upgrade head (через образ)
make prod-up                # api + worker
make prod-health            # /healthz /readyz на :8000
```

### 9.3. Parser infrastructure + app

```bash
cd /opt/markethacker/parser
make infra-up-prod          # postgres, pgbouncer, redis, clickhouse, kafka
make prod-migrate           # PG + ClickHouse
make prod-up
curl -fsS http://127.0.0.1:8010/api/v1/readyz
```

Проверка связи backend → parser:

```bash
cd /opt/markethacker/backend
make prod-check-parser
```

### 9.4. Frontends + landing

```bash
cd /opt/markethacker/admin-panel
make prod-up && make prod-health

cd /opt/markethacker/manager-portal
make prod-up && make prod-health

cd /opt/markethacker/market-navigators
docker compose -f docker-compose.prod.yml up -d --build
curl -fsS http://127.0.0.1:3003/health
```

### 9.5. Caddy (TLS)

DNS уже должен указывать на VPS.

```bash
cd /opt/markethacker/caddy
make validate
make up
make health
```

Проверка снаружи:

```bash
curl -fsS https://api.markethacker.ru/healthz
curl -fsSI https://markethacker.ru/ | head
curl -fsSI https://admin.markethacker.ru/login | head
curl -fsSI https://team.markethacker.ru/login | head
```

### 9.6. Monitoring

Нужны сети `markethacker_backend_infra` / `markethacker_parser_infra` (exporters) и поднятые backend/parser.

```bash
cd /opt/markethacker/monitoring
make up
make health
```

Откройте `https://grafana.markethacker.ru` (логин из `.env`).  
Дашборды: Overview, Backend, Parser, Workers, Docker, Infra, Availability, PostgreSQL, Redis, ClickHouse, Kafka, Business.  
Prometheus UI: `ssh -L 9090:127.0.0.1:9090 ubuntu@vps` → http://127.0.0.1:9090.  
Alertmanager: `ssh -L 9093:127.0.0.1:9093 …`.

### 9.7. Первый суперпользователь

Зарегистрируйте пользователя через API/портал, затем:

```bash
cd /opt/markethacker/backend
docker compose -f docker-compose.prod.yml run --rm api \
  python scripts/promote-superuser.py you@example.com
```

(если скрипт не скопирован в образ — выполните через `uv` на хосте с `DATABASE_DIRECT_URL`, либо добавьте `scripts/` в Dockerfile; на текущем образе проверьте наличие файла).

---

## 10. Базы данных — детали

### Порядок

1. Сети  
2. Backend PG/Redis  
3. Parser PG/Redis/CH/Kafka  
4. Миграции  
5. Apps  

Пользователи/БД создаются переменными `POSTGRES_*` при первом старте контейнера postgres (официальный entrypoint). Отдельно `CREATE USER` не требуется.

### Хранение

Docker named volumes (`pgdata`, `redisdata`, `clickhousedata`, `kafkadata`, `caddy_data`, `prometheus_data`, `grafana_data`, `markethacker_support_attachments`).  
Вложения чата поддержки живут в `markethacker_support_attachments` (пока не настроен `SUPPORT_S3_*`).  
Список: `docker volume ls | grep markethacker`.

### Backup

```bash
cd /opt/markethacker/infra
make backup
# cron от ubuntu:
crontab -e
# 15 3 * * * /opt/markethacker/infra/scripts/backup-all.sh >> /opt/markethacker/backups/backup.log 2>&1
```

Скрипт кладёт `pg_dump` (backend+parser), ClickHouse `BACKUP DATABASE`, снимки Redis, копии `.env` и Caddyfile в `/opt/markethacker/backups/<timestamp>.tar.gz`, ротация 14 дней.

**Обязательно** копируйте архивы offsite (Object Storage / другой сервер) — бэкап на том же диске не спасает от потери VPS.

---

## 11. Миграции

| Когда | Как |
|-------|-----|
| Первый деплой | `make prod-migrate` до или вместе с `prod-up` |
| Обновление | всегда **до** переключения трафика на новый код: migrate → up |
| Ошибка | смотреть `docker compose … logs`, не запускать app со старой схемой; исправить миграцию/данные; при необходимости `alembic downgrade -1` (`make prod-rollback` в backend) |
| Успех PG | `docker compose -f docker-compose.prod.yml run --rm api alembic current` |
| Успех CH | `… run --rm api python -m markethacker_parser.cli clickhouse status` |

Миграции backend идут через **прямой** PostgreSQL (`DATABASE_DIRECT_URL` / hostname `postgres`), не через transaction pooling нюансы — в образе/compose это учтено через env.

---

## 12. Reverse proxy и SSL

Caddy:

- автоматический HTTPS (Let's Encrypt);
- HSTS `max-age=31536000; includeSubDomains; preload`;
- security headers;
- `encode gzip zstd`;
- `/metrics` API защищён Basic Auth;
- `wb-proxy` делает rewrite на `/api/v1/wb-gateway`;
- `wb-connect` и onboarding-поддомены — **без** rewrite.

Продление сертификатов — встроенный ACME Caddy (не cron certbot).  
Проверка: `docker compose exec caddy caddy list-modules` / логи при старте; браузерный замок; `echo | openssl s_client -connect api.markethacker.ru:443 -servername api.markethacker.ru 2>/dev/null | openssl x509 -noout -dates`.

Systemd для приложений **не нужен**: всё в Docker с `restart: unless-stopped`, Docker сам поднимается через `systemd` unit `docker.service`.

---

## 13. Логирование

```bash
docker compose -f docker-compose.prod.yml logs -f --tail=200 api
docker logs markethacker-caddy --tail=200
```

Ротация: daemon.json + per-service `max-size: 10m`.  
Приложения пишут structured JSON (structlog) в stdout.

---

## 14. Мониторинг — что считается «живым»

| Проверка | Команда / URL |
|----------|----------------|
| Backend liveness | `curl -fsS https://api.markethacker.ru/healthz` |
| Backend ready | `curl -fsS https://api.markethacker.ru/readyz` |
| Parser | `curl -fsS http://127.0.0.1:8010/api/v1/readyz` |
| Admin | `curl -fsS -o /dev/null -w '%{http_code}\n' https://admin.markethacker.ru/login` |
| Team | то же для `team.markethacker.ru` |
| Landing | `https://markethacker.ru/` |
| Контейнеры | `docker ps --format 'table {{.Names}}\t{{.Status}}'` |
| Диск/RAM | Grafana dashboard «MarketHacker Overview» + алерты Prometheus |
| Сертификаты | blackbox `fail_if_not_ssl` + ручная проверка дат |
| Метрики API | `https://api.markethacker.ru/metrics` (Basic Auth) и scrape Prometheus |

---

## 15. Безопасное обновление

```bash
# пример backend
cd /opt/markethacker/backend
git fetch origin --tags
git checkout main
git reset --hard vX.Y.Z          # или origin/main

make prod-migrate
make prod-up
make prod-health
```

Аналогично parser / frontends / caddy / monitoring.

**Rollback приложения (без миграции вниз):**

```bash
git reset --hard vPREV
FULL_IMAGE=markethacker-api:PREV docker compose -f docker-compose.prod.yml up -d --build
```

**Rollback миграции** — только если миграция обратима и согласована: `make prod-rollback` (одна ревизия). Имейте свежий `pg_dump` до опасных миграций.

Опционально позже можно подключить GitHub Actions (`DEPLOY_USER=ubuntu`, `DEPLOY_PATH=/opt/markethacker/backend`) — для первого поднятия и повседневных обновлений достаточно шагов выше по SSH.
---

## 16. Чеклист после деплоя

- [ ] `docker ps` — все сервисы `healthy` / `Up`
- [ ] HTTPS на api/admin/team/landing/grafana
- [ ] `make prod-check-parser`
- [ ] Логин в admin-panel (superuser)
- [ ] Логин в manager-portal
- [ ] Volume `markethacker_support_attachments` есть (`docker volume ls | grep support`) — если без S3
- [ ] Webhook ЮKassa/Stripe указывают на `https://api.markethacker.ru/api/v1/billing/webhooks/...`
- [ ] Grafana: Overview + Backend/Workers/Postgres targets UP
- [ ] Alertmanager: test alert → Telegram (или proxy/Bot API)
- Worker metrics: `curl -fsS http://127.0.0.1:9101/metrics | head` / `:9102`
- ClickHouse metrics: `curl -fsS http://127.0.0.1:9363/metrics | head`
- [ ] `make backup` отрабатывает без ошибок
- [ ] fail2ban / ufw активны
- [ ] Root SSH login отключён

---

## 17. Типичные ошибки

### `Permission denied` при `git clone` в `/opt/...`

**Причина:** каталог принадлежит root.  
**Решение:** `sudo chown -R ubuntu:ubuntu /opt/markethacker` (раздел 2).

### `Permission denied (publickey)` от GitHub

**Причина:** на VPS нет SSH-ключа или ключ не добавлен в ваш GitHub-аккаунт.  
**Диагностика:** `ssh -T git@github.com`.  
**Решение:** раздел 5 — создать ключ у `ubuntu`, добавить `.pub` в GitHub → SSH keys.
### Caddy не получает сертификат

**Причина:** DNS не на VPS, закрыт :80, другой процесс на 80/443.  
**Диагностика:** `docker logs markethacker-caddy`, `ss -tlnp | grep -E ':80|:443'`.  
**Решение:** исправить DNS/UFW, остановить конфликтующий nginx на хосте.

### Backend `readyz` degraded

**Причина:** PG/Redis не доступны или неверный `REDIS_URL` без пароля.  
**Диагностика:** `docker compose -f docker-compose.yml ps`, логи redis (`NOAUTH`).  
**Решение:** задать `REDIS_PASSWORD` и `REDIS_URL=redis://:pass@redis:6379/0`, пересоздать redis.

### `network markethacker_* not found`

**Причина:** не выполнен bootstrap.  
**Решение:** `cd /opt/markethacker/infra && make networks`.

### Backend не видит parser

**Причина:** parser API не в `markethacker_apps` или нет `container_name`.  
**Решение:** `make prod-up` в parser; `make prod-check-parser` в backend.

### ClickHouse / Kafka OOM (exit 137 / 36)

**Причина:** мало RAM.  
**Решение:** увеличить VPS; снизить `KAFKA_HEAP_OPTS`; не ставить жёсткий `mem_limit: 2g` на CH.

### `NEXT_PUBLIC_API_URL` указывает на localhost в проде

**Причина:** забыли `.env` / не пересобрали образ.  
**Решение:** исправить `.env`, `make prod-up` (с `--build`).

### Grafana пустая / datasource down

**Причина:** Prometheus не слушает или scrape targets down.  
**Диагностика:** `curl http://127.0.0.1:9090/api/v1/targets`.  
**Решение:** поднять backend/parser; проверить `network_mode: host`.

---

## 18. Карта репозиториев

| Repo | Remote | Путь на VPS |
|------|--------|-------------|
| infra | MarketHackerLabs/infra | `/opt/markethacker/infra` |
| backend | MarketHackerLabs/backend | `/opt/markethacker/backend` |
| parser | MarketHackerLabs/parser | `/opt/markethacker/parser` |
| admin-panel | MarketHackerLabs/admin-panel | `/opt/markethacker/admin-panel` |
| manager-portal | MarketHackerLabs/manager-portal | `/opt/markethacker/manager-portal` |
| caddy | MarketHackerLabs/caddy | `/opt/markethacker/caddy` |
| monitoring | MarketHackerLabs/monitoring | `/opt/markethacker/monitoring` |
| market-navigators | (см. remote в git) | `/opt/markethacker/market-navigators` |

После создания GitHub-репозитория `MarketHackerLabs/monitoring` запушьте локальный каталог `monitoring/`.

---

## 19. Что было улучшено относительно «как было в коде»

- Канон путей `/opt/markethacker/...`
- Non-root в backend/parser Docker images (uid 10001)
- Обязательный пароль Redis
- Исправлен Makefile manager-portal
- Endpoint `/metrics` (Prometheus) в backend и parser
- Репозиторий monitoring + домен `grafana.markethacker.ru`
- Скрипт бэкапов в infra

Держите эту инструкцию рядом с репозиторием `docs` и обновляйте при смене топологии.
