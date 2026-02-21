# Пошаговый деплой: локально → GitHub → сервер

Сборка проекта на локальном диске, выкладка на GitHub и установка на сервере (включая miniapp на bot.ghostvpn.cc).

---

## Часть 1. Сборка на локальном диске

### Шаг 1.1. Проверь структуру проекта

В корне проекта `ghostvpnbot` должны быть:

- `app/`, `docs/`, `migrations/`, `locales/` — код и ресурсы
- `miniapp/` — папка miniapp с `index.html` и `app-config.json`
- `deploy/` — пример конфига nginx для bot.ghostvpn.cc
- `.env.example`, `docker-compose.yml`, `Dockerfile`, `main.py`

Если папок `miniapp/` или `deploy/` нет — они уже добавлены в репозиторий; обнови файлы из репо (`git pull` или скопируй из текущего состояния проекта).

### Шаг 1.2. Локальный .env (не коммитить)

Для локальной проверки (по желанию):

```powershell
copy .env.example .env
# Открой .env и заполни минимум: BOT_TOKEN, ADMIN_IDS, REMNAWAVE_API_URL, REMNAWAVE_API_KEY
```

Файл `.env` в репозиторий не попадает (он в `.gitignore`).

### Шаг 1.3. Убедись, что в коммит не попадут лишние файлы

Проверка в корне проекта:

```powershell
git status
```

В списке не должно быть `.env`. Должны быть видны изменения в `miniapp/`, `deploy/`, `docs/`, `.gitignore` (если ты их менял).

### Шаг 1.4. Что будет залито на GitHub

- Весь код: `app/`, `tests/`, `migrations/`, `locales/`, `docs/`, `assets/`, `.github/`
- Конфиги: `.env.example`, `docker-compose.yml`, `Dockerfile`, `Makefile`, `pyproject.toml`, `requirements.txt`, `alembic.ini` и т.д.
- Miniapp: `miniapp/index.html`, `miniapp/app-config.json`
- Deploy: `deploy/nginx-bot.ghostvpn.cc.conf.example`
- Документация: `README.md`, `docs/DEPLOY_SERVER_AND_GIT.md`, `docs/MINIAPP_BOT_GHOSTVPN_CC.md`, `docs/DEPLOY_FULL_STEPS.md` (этот файл)

Не попадут: `.env`, `logs/`, `data/`, `*.log`, виртуальные окружения.

---

## Часть 2. Выкладка на GitHub

### Шаг 2.1. Закоммитить и отправить

В корне проекта выполни:

```powershell
git add .
git status
```

Проверь список: не должно быть `.env`. Затем:

```powershell
git commit -m "Add miniapp, deploy config and full deploy docs"
git push origin main
```

Если ветка называется иначе (например `master`), подставь её имя. Если remote ещё не настроен:

```powershell
git remote add origin https://github.com/ТВОЙ_ЛОГИН/ghostvpnbot.git
git push -u origin main
```

### Шаг 2.2. Проверка на GitHub

Зайди в репозиторий на GitHub и убедись, что есть:

- папка `miniapp/` с `index.html` и `app-config.json`;
- папка `deploy/` с примером nginx;
- актуальные `docs/`.

После этого переходи к установке на сервере.

---

## Часть 3. Установка на сервере

Сервер: Linux с установленными **Docker**, **Docker Compose**, **Git** и (для miniapp) **Nginx** и **certbot**.

### Шаг 3.1. Клонирование и подготовка

На сервере:

```bash
cd ~
git clone https://github.com/mentyrop/ghostvpnbot.git
cd ghostvpnbot
```

Создай `.env` из примера и заполни под свой сервер:

```bash
cp .env.example .env
nano .env
```

Обязательно заполни:

- `BOT_TOKEN`, `ADMIN_IDS`
- `POSTGRES_PASSWORD` (надёжный пароль)
- `REMNAWAVE_API_URL`, `REMNAWAVE_API_KEY` (панель на этом сервере или доступная по сети)
- Для webhook: `BOT_RUN_MODE=webhook`, `WEBHOOK_URL`, `WEBHOOK_PATH`, `WEBHOOK_SECRET_TOKEN`
- Для miniapp: `WEB_API_ENABLED=true`, `WEB_API_ALLOWED_ORIGINS=https://bot.ghostvpn.cc`, `WEB_API_DEFAULT_TOKEN=<openssl rand -hex 32>`, `CONNECT_BUTTON_MODE=miniapp_subscription`

### Шаг 3.2. Запуск бота (Docker)

Контейнер бота работает от пользователя с UID 1000. Папки `logs` и `data` на хосте должны быть ему доступны на запись, иначе будет `Permission denied: '/app/logs/bot.log'`.

```bash
mkdir -p ./logs ./data ./data/backups ./data/referral_qr
chmod -R 755 ./logs ./data
# Контейнер бота работает от UID 1000 — папки должны быть ему доступны на запись
sudo chown -R 1000:1000 ./logs ./data ./locales
docker compose up -d --build
docker compose logs -f bot
```

Убедись, что бот стартует без ошибок. Проверка здоровья API (если включён):

```bash
curl -s -H "X-API-Key: ВАШ_WEB_API_DEFAULT_TOKEN" http://127.0.0.1:8080/health
```

### Шаг 3.3. Nginx и miniapp на bot.ghostvpn.cc

1. Установи Nginx и certbot (если ещё не стоят):

```bash
sudo apt update && sudo apt install -y nginx certbot python3-certbot-nginx
```

2. SSL-сертификат для bot.ghostvpn.cc (сначала Nginx не должен занимать 80 порт для этого домена, или используй certbot в режиме standalone/webroot по инструкции):

```bash
sudo certbot certonly --standalone -d bot.ghostvpn.cc --agree-tos --email твой@email.com --non-interactive
```

Либо через плагин nginx после добавления конфига с `server_name bot.ghostvpn.cc`:

```bash
sudo certbot --nginx -d bot.ghostvpn.cc --agree-tos --email твой@email.com --non-interactive
```

3. Конфиг Nginx для bot.ghostvpn.cc:

Скопируй пример и подставь путь к папке miniapp (на сервере это каталог внутри клонированного репо):

```bash
sudo cp deploy/nginx-bot.ghostvpn.cc.conf.example /etc/nginx/sites-available/bot.ghostvpn.cc
sudo nano /etc/nginx/sites-available/bot.ghostvpn.cc
```

Замени строку с `root` на реальный путь, например:

- `root /home/ubuntu/ghostvpnbot/miniapp;` (подставь своего пользователя и путь к репо).

В блоке `location /miniapp/` оставь `proxy_pass http://127.0.0.1:8080/miniapp/;` если бот слушает на том же сервере на порту 8080.

4. Включи сайт и перезагрузи Nginx:

```bash
sudo ln -sf /etc/nginx/sites-available/bot.ghostvpn.cc /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

5. Проверка в браузере: открой `https://bot.ghostvpn.cc` — должна открыться страница miniapp (вне Telegram может быть сообщение «Откройте из бота» или загрузка).

### Шаг 3.4. Nginx для платёжных касс (webhook'и на домене)

Чтобы кассы (ЮKassa, CloudPayments и др.) могли слать уведомления на бота, нужен URL вида **https://bot.ghostvpn.cc/yookassa-webhook** (домен не ниже второго уровня, не IP:port). Ниже — по порядку, что сделать на сервере.

**Шаг 3.4.1. Подключись к серверу и перейди в каталог проекта**

```bash
ssh root@ТВОЙ_IP
# или: ssh ubuntu@ТВОЙ_IP
cd ~/ghostvpnbot
```

**Шаг 3.4.2. Обнови репозиторий (если конфиг nginx хранится в репо)**

```bash
git pull origin main
```

**Шаг 3.4.3. Скопируй пример конфига Nginx в sites-available**

```bash
sudo cp deploy/nginx-bot.ghostvpn.cc.conf.example /etc/nginx/sites-available/bot.ghostvpn.cc
```

Если файл `/etc/nginx/sites-available/bot.ghostvpn.cc` уже есть и ты его раньше правил вручную — не перезаписывай, а открой его для редактирования и добавь блок webhook'ов из примера (см. шаг 3.4.4).

**Шаг 3.4.4. Открой конфиг и поправь путь к miniapp**

```bash
sudo nano /etc/nginx/sites-available/bot.ghostvpn.cc
```

- Найди строку `root /путь/к/ghostvpnbot/miniapp;`.
- Замени на реальный путь к папке miniapp на сервере, например:
  - `root /root/ghostvpnbot/miniapp;` (если заходишь под root и проект в /root/ghostvpnbot)
  - или `root /home/ubuntu/ghostvpnbot/miniapp;` (если пользователь ubuntu и проект в домашней папке)
- Убедись, что в конфиге есть блок для webhook'ов (проксирование на 8080):

```nginx
    location ~ ^/(webhook|.+-webhook)(/|$) {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
    }
```

Если этого блока нет — добавь его внутри `server { ... }` для 443, **выше** блока `location /`. Сохрани файл: в nano — `Ctrl+O`, Enter, затем `Ctrl+X`.

**Шаг 3.4.5. Проверь пути к SSL-сертификатам**

В том же файле должны быть строки:

- `ssl_certificate     /etc/letsencrypt/live/bot.ghostvpn.cc/fullchain.pem;`
- `ssl_certificate_key /etc/letsencrypt/live/bot.ghostvpn.cc/privkey.pem;`

Если сертификат у тебя для другого домена или в другом пути — замени на свои. Проверка наличия файлов:

```bash
sudo ls -la /etc/letsencrypt/live/bot.ghostvpn.cc/
```

**Шаг 3.4.6. Включи сайт и проверь конфигурацию Nginx**

```bash
sudo ln -sf /etc/nginx/sites-available/bot.ghostvpn.cc /etc/nginx/sites-enabled/
sudo nginx -t
```

Ожидается строка: `syntax is ok` и `test is successful`. Если есть ошибки — исправь конфиг по сообщениям.

**Шаг 3.4.7. Применить конфиг (перезагрузить Nginx)**

```bash
sudo systemctl reload nginx
```

**Шаг 3.4.8. Проверить, что бот слушает 8080 и webhook отвечает**

Убедись, что контейнер бота запущен:

```bash
cd ~/ghostvpnbot
docker compose ps
```

Должен быть запущен сервис `remnawave_bot`. Затем проверь, что через домен до webhook'а доходят запросы (например, ЮKassa часто делает GET для проверки):

```bash
curl -s -o /dev/null -w "%{http_code}" https://bot.ghostvpn.cc/yookassa-webhook
```

Код ответа может быть 200, 405 (Method Not Allowed для GET — для ЮKassa норма) или другой не 502/504. Главное — не 502 Bad Gateway (значит nginx до бота не доходит).

**Шаг 3.4.9. Что указывать в личном кабинете кассы**

В настройках платёжной системы укажи URL на домене, например:

- ЮKassa: `https://bot.ghostvpn.cc/yookassa-webhook`
- CloudPayments: `https://bot.ghostvpn.cc/cloudpayments-webhook`
- FreeKassa: `https://bot.ghostvpn.cc/freekassa-webhook`

и т.д. по списку из проекта. Не используй `http://IP:8080/...`.

**Кратко по порядку:** (1) зайти на сервер и в каталог проекта, (2) при необходимости обновить репо, (3) скопировать/обновить конфиг nginx, (4) поправить `root` и при отсутствии — добавить блок webhook'ов, (5) проверить SSL-пути, (6) включить сайт и `nginx -t`, (7) `systemctl reload nginx`, (8) проверить ответ webhook'а через curl, (9) в кассе прописать URL вида `https://bot.ghostvpn.cc/...-webhook`.

**Шаг 3.4.10. Один порт 443 для бота и панели**

Если на том же сервере стоит панель в Docker и она занимает 443, nginx не сможет стартовать. Настрой один nginx на 443 с маршрутизацией по домену:

1. **Панель:** в `docker-compose` панели замени проброс порта с `443:443` (или `443:80`) на **только localhost**, например:
   ```yaml
   ports:
     - "127.0.0.1:8443:443"   # или 8443:80, если внутри контейнера слушает 80
   ```
   Перезапусти контейнеры панели (`docker compose up -d` в каталоге панели). Порт 443 на хосте освободится.

2. **Nginx:** используй объединённый конфиг из `deploy/nginx-bot.ghostvpn.cc.conf.example` — в нём два блока: для `bot.ghostvpn.cc` (прокси на 127.0.0.1:8080) и для домена панели (прокси на 127.0.0.1:8443). В конфиге замени:
   - `PANEL_DOMAIN` на реальный домен панели (например `panel.ghostvpn.cc`);
   - пути к SSL-сертификату панели: `/etc/letsencrypt/live/PANEL_DOMAIN/...` на свои (при необходимости получи сертификат: `sudo certbot certonly -d panel.ghostvpn.cc`).

3. Убедись, что в `sites-enabled` нет конфига `default`, который слушает 443. Перезагрузи nginx: `sudo nginx -t && sudo systemctl reload nginx`. Проверь: `https://bot.ghostvpn.cc` и `https://домен-панели`.

**Шаг 3.4.11. Бот на порту 8443 (панель на 443)**

Если панель остаётся на 443 и не трогаешь её, вебхуки и miniapp бота можно повесить на отдельный порт **8443**:

1. Используй конфиг `deploy/nginx-bot-port-8443.conf.example`: в нём один `server` на порту **8443** для `bot.ghostvpn.cc` (прокси на 127.0.0.1:8080). Замени путь `root` на путь к miniapp на сервере.
2. Скопируй в `/etc/nginx/sites-available/bot.ghostvpn.cc`, включи через `sites-enabled`, открой порт в файрволе: `sudo ufw allow 8443/tcp && sudo ufw reload`.
3. Запусти nginx: `sudo nginx -t && sudo systemctl start nginx && sudo systemctl enable nginx`.
4. В кассе указывай webhook: `https://bot.ghostvpn.cc:8443/yookassa-webhook` (и т.д.). Miniapp в BotFather: `https://bot.ghostvpn.cc:8443`.

Часть касс не поддерживает нестандартный порт в URL; тогда единственный вариант — один nginx на 443 (шаг 3.4.10).

### Шаг 3.5. Настройка бота и BotFather

1. В админке бота (в Telegram): **Конфигурации бота → Прочее → Miniapp** — укажи URL miniapp: `https://bot.ghostvpn.cc`.
2. Перезапусти бота: `docker compose restart bot`.
3. При необходимости в **@BotFather**: `/setmenu` → Web App URL: `https://bot.ghostvpn.cc`.

---

## Часть 4. Дальнейшие обновления

- **Локально:** правки → `git add .` → `git commit -m "..."` → `git push origin main`.
- **На сервере:**  
  `cd ~/ghostvpnbot` → `git pull origin main` → `docker compose up -d --build` (или `docker compose restart bot`).

Подробнее: [DEPLOY_SERVER_AND_GIT.md](DEPLOY_SERVER_AND_GIT.md). Детали miniapp и диагностика: [MINIAPP_BOT_GHOSTVPN_CC.md](MINIAPP_BOT_GHOSTVPN_CC.md).

---

## Часть 5. Ошибка при pull образов (Docker Hub / IPv6)

Если при `docker compose up` появляется ошибка вида:

```text
failed to resolve reference "docker.io/library/postgres:15-alpine"
dial tcp [2a00:...]:443: connect: cannot assign requested address
```

это сетевая проблема: до Docker Hub нет доступа или не работает IPv6.

**Что сделать на сервере:**

1. **Заставить Docker использовать IPv4** (часто помогает):

   ```bash
   sudo mkdir -p /etc/docker
   echo '{"ipv6": false}' | sudo tee /etc/docker/daemon.json
   sudo systemctl restart docker
   ```

   Затем снова:

   ```bash
   cd ~/ghostvpnbot
   docker compose pull
   docker compose up -d --build
   ```

2. **Проверить доступ в интернет к Docker Hub:**

   ```bash
   curl -v --connect-timeout 5 https://registry-1.docker.io/v2/
   ```

   Если запрос не проходит — на сервере или у хостера блокируют/режут доступ к Docker Hub. Тогда остаётся:
   - использовать VPN/прокси на сервере;
   - или зеркало (registry mirror) в `daemon.json`, если у хостера есть инструкция.

3. **Если образы уже есть на другой машине** — можно сохранить их в файл и загрузить на сервер:

   На машине, где pull работает:
   ```bash
   docker save postgres:15-alpine redis:7-alpine -o ghostvpnbot-images.tar
   ```
   Передать `ghostvpnbot-images.tar` на сервер (scp, rsync и т.п.), затем на сервере:
   ```bash
   docker load -i ghostvpnbot-images.tar
   docker compose up -d --build
   ```

---

## Часть 6. Ошибки «Permission denied» для locales и «Name or service not known» для БД

**1. Локали (Locale directory is not writable)**  
Контейнер пишет в `/app/locales` (это твоя папка `./locales`). Отдай владение пользователю 1000:

```bash
sudo chown -R 1000:1000 ./locales
docker compose restart bot
```

**2. База данных (gaierror: Name or service not known)**  
Бот не может разрешить имя хоста PostgreSQL. В `docker-compose.yml` для сервиса bot уже заданы **POSTGRES_HOST=172.20.0.2** и **DATABASE_URL** с этим IP, чтобы не зависеть от DNS.

- Убедись, что на сервере актуальный `docker-compose.yml` из репо (после `git pull`). В нём у сервиса `bot` в `environment` должны быть `POSTGRES_HOST: '172.20.0.2'` и строка `DATABASE_URL: 'postgresql+asyncpg://...@172.20.0.2:5432/...'`.
- В `.env` на сервере задай **POSTGRES_PASSWORD** (и при необходимости POSTGRES_USER/POSTGRES_DB). Строку **DATABASE_URL** в `.env` лучше не задавать — тогда подставится URL из compose с IP. Если пароль содержит символы **@** или **#**, либо смени пароль, либо в compose закомментируй строку `DATABASE_URL: '...'` — тогда приложение соберёт URL из POSTGRES_HOST и POSTGRES_PASSWORD.
- Пересоздай контейнеры, чтобы подхватить переменные:
  ```bash
  docker compose down
  docker compose up -d --build
  docker compose logs -f bot
  ```
- Проверка, что контейнер бота видит хост по IP:
  ```bash
  docker compose exec bot env | grep -E '^POSTGRES_HOST=|^DATABASE_URL='
  ```
  Должно быть `POSTGRES_HOST=172.20.0.2` и `DATABASE_URL=postgresql+asyncpg://...@172.20.0.2:5432/...` (или только POSTGRES_HOST, если DATABASE_URL закомментирован).

- **Если БД внешняя** (отдельный сервер): в `.env` укажи свой хост в `POSTGRES_HOST` или в `DATABASE_URL`. С сервера хост должен резолвиться: `getent hosts ИМЯ_ХОСТА` и `nc -zv ИМЯ_ХОСТА 5432`.

**3. Чистый пересоздание (если ошибка «Name or service not known» не исчезает)**  
Чтобы убрать влияние старого состояния (раньше крутили локально, потом перенесли на сервер):

1. На сервере обнови код и убедись, что в compose заданы хост по IP и DATABASE_URL:
   ```bash
   cd ~/ghostvpnbot
   git pull origin main
   grep -A2 POSTGRES_HOST docker-compose.yml
   ```
2. В `.env` задай только нужные переменные (BOT_TOKEN, ADMIN_IDS, **POSTGRES_PASSWORD** и т.д.). Не задавай `DATABASE_URL=` с хостом `postgres` или другим именем.
3. Останови всё и удали тома (**данные БД будут удалены**):
   ```bash
   docker compose down -v
   ```
4. Запусти заново:
   ```bash
   docker compose up -d --build
   docker compose logs -f bot
   ```
5. Если после этого ошибка остаётся — проверь, что внутри контейнера реально подхватился URL с IP:
   ```bash
   docker compose run --rm --no-deps bot env | grep -E 'POSTGRES_HOST|DATABASE_URL'
   ```
   Должно быть `172.20.0.2` в одной из переменных.
