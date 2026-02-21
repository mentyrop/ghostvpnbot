# Деплой на сервер и работа через Git

Инструкция: как выложить код на GitHub, развернуть бота на сервере и вносить правки локально с обновлением на сервере через Git.

---

## 1. Подготовка кода и отправка на GitHub

### Убедись, что `.env` не попадёт в репозиторий

Файл `.env` уже в `.gitignore` — в репозиторий он не коммитится. На сервере ты создашь свой `.env` заново.

Проверка (в корне проекта):

```bash
git status
```

В списке не должно быть `.env`. Если он есть — не добавляй его в коммит.

### Пуш кода на GitHub

```bash
git add .
git status
git commit -m "Initial: GhostVPN bot"
git remote get-url origin
git push -u origin main
```

Если `origin` ещё не твой репозиторий:

```bash
git remote set-url origin https://github.com/ТВОЙ_ЛОГИН/ghostvpnbot.git
git push -u origin main
```

После этого весь код (без `.env`) будет на GitHub.

---

## 2. Деплой на сервер (первый раз)

Сервер должен быть с установленными **Docker** и **Docker Compose** (и Git).

### 2.1 Клонирование репозитория

На сервере (в той папке, где хочешь проект, например `~/ghostvpnbot`):

```bash
git clone https://github.com/ТВОЙ_ЛОГИН/ghostvpnbot.git
cd ghostvpnbot
```

### 2.2 Создание и настройка `.env`

Так как сервер другой — `.env` нужно создать заново и заполнить под новый хостинг.

```bash
cp .env.example .env
nano .env
```

Минимум что нужно заполнить под новый сервер:

| Переменная | Описание |
|------------|----------|
| `BOT_TOKEN` | Токен бота от @BotFather |
| `ADMIN_IDS` | Твой Telegram ID (или список через запятую) |
| `ADMIN_NOTIFICATIONS_CHAT_ID` | ID канала/чата для уведомлений (если нужны) |
| `POSTGRES_PASSWORD` | Надёжный пароль БД (на сервере свой) |
| `REMNAWAVE_API_URL` | URL панели на **новом** сервере |
| `REMNAWAVE_API_KEY` | Ключ API панели на новом сервере |
| При webhooks: `REMNAWAVE_WEBHOOK_SECRET`, публичный URL бота для вебхуков |

Остальное — по необходимости (платежи, SMTP, каналы и т.д.). В `.env.example` есть комментарии.

### 2.3 Запуск

```bash
docker compose up -d --build
```

Проверка логов:

```bash
docker compose logs -f bot
```

Бот должен стартовать; БД и Redis поднимаются через `docker-compose.yml`.

---

## 3. Рабочий процесс: правки локально и обновление на сервере

### Локально (у себя на компьютере)

1. Вносишь изменения в код.
2. Коммит и пуш:

```bash
git add .
git commit -m "Описание правок"
git push origin main
```

### На сервере (после твоего push)

Обновить код и перезапустить бота:

```bash
cd ~/ghostvpnbot
git pull origin main
docker compose up -d --build
```

- `git pull` — подтягивает последние коммиты с GitHub.
- `docker compose up -d --build` — пересобирает образ (если менялся код/Dockerfile) и перезапускает контейнеры.

Если менялся только код приложения (без зависимостей и Dockerfile), часто достаточно:

```bash
git pull origin main
docker compose restart bot
```

Так ты всегда работаешь с одним и тем же Git-репозиторием: правки локально → push → на сервере pull и перезапуск.

---

## 4. Смена сервера (переезд)

Когда снова меняешь сервер:

1. На **новом** сервере: клонируй репозиторий заново (`git clone ...`), создай новый `.env` из `.env.example` и заполни под новый хостинг (URL панели, пароли, webhook URL и т.д.).
2. Запусти: `docker compose up -d --build`.
3. Старый сервер можешь отключить; `.env` с него в репо не нужен — на новом сервере свой `.env`.

Код один и тот же с GitHub; различаются только настройки в `.env` на каждом сервере.

---

## 5. Запуск WebApp с твоим доменом

В проекте есть два варианта «веба» для пользователей.

### Зачем это нужно

- **Miniapp** — страница прямо в Telegram (кнопка «Подключиться» открывает WebApp): подписка, ссылки на конфиги, иногда оплата. Удобно: пользователь не ходит по шагам в боте, всё в одном окне.
- **Cabinet** — полноценный личный кабинет (отдельный фронт на своём домене): вход по Telegram или email, управление подпиской, продление, рефералы. Нужен, если хочешь полноценный ЛК в браузере.

Если достаточно показать подписку и кнопки подключения в Telegram — хватает **Miniapp**. Если нужен отдельный сайт-кабинет — подключаешь **Cabinet**.

---

### Вариант A: Miniapp (встроенный в бота)

Домен у тебя уже есть — вешаешь на него статику и прокси к боту.

1. **В `.env` на сервере** (подставь свой домен):

```env
WEB_API_ENABLED=true
WEB_API_HOST=0.0.0.0
WEB_API_PORT=8080
WEB_API_ALLOWED_ORIGINS=https://ТВОЙ_ДОМЕН.com
WEB_API_DEFAULT_TOKEN=сгенерируй_openssl_rand_hex_32
```

И для кнопки «Подключиться» в Telegram:

```env
CONNECT_BUTTON_MODE=miniapp_subscription
```

2. **Nginx** (или другой прокси) на сервере:
   - Раздавать статику из папки `miniapp/` (если есть в репо) по корню домена или по пути `/miniapp`.
   - Проксировать запросы `/miniapp/*` на бота: `http://127.0.0.1:8080/miniapp/` (порт 8080 — как в `docker-compose`, если бот на том же сервере).

Пример для Nginx (замени `miniapp.твой-домен.com` и пути на свои):

```nginx
server {
    listen 443 ssl http2;
    server_name miniapp.твой-домен.com;
    # ssl_certificate / путь/к/fullchain.pem;
    # ssl_certificate_key / путь/к/privkey.pem;

    root /путь/к/ghostvpnbot/miniapp;
    index index.html;

    location /miniapp/ {
        proxy_pass http://127.0.0.1:8080/miniapp/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        try_files $uri /index.html =404;
    }
}
```

3. В **@BotFather** можно задать меню с Web App: команда `/setmenu` → указать URL твоего домена (HTTPS), чтобы кнопка «Открыть» открывала miniapp.

Подробнее: [docs/miniapp-setup.md](miniapp-setup.md).

---

### Вариант B: Cabinet (полноценный ЛК)

Отдельный репозиторий: [BEDOLAGA-DEV/bedolaga-cabinet](https://github.com/BEDOLAGA-DEV/bedolaga-cabinet). Разворачиваешь фронт на своём домене (например `cabinet.твой-домен.com`), бот отдаёт API.

В `.env` бота:

```env
CABINET_ENABLED=true
CABINET_URL=https://cabinet.твой-домен.com
CABINET_ALLOWED_ORIGINS=https://cabinet.твой-домен.com
CABINET_JWT_SECRET=сгенерируй_длинный_секрет
```

И настрой CORS/прокси так, чтобы запросы с домена кабинета шли к API бота (порт 8080). Детали — в README репозитория bedolaga-cabinet.

---

**Итог:** да, webapp полезен: пользователи видят подписку и подключаются из одного окна в Telegram (miniapp) или из браузера (cabinet). С твоим уже развёрнутым доменом достаточно прописать в `.env` свой домен в `WEB_API_ALLOWED_ORIGINS`, выставить `CONNECT_BUTTON_MODE=miniapp_subscription` и настроить Nginx по примеру выше.
