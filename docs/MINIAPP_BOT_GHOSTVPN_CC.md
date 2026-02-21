# Miniapp на bot.ghostvpn.cc

Пошаговая настройка мини-приложения по инструкциям из репозитория [BEDOLAGA-DEV/remnawave-bedolaga-telegram-bot](https://github.com/BEDOLAGA-DEV/remnawave-bedolaga-telegram-bot) и [docs/miniapp-setup.md](miniapp-setup.md). Домен: **bot.ghostvpn.cc**.

---

## 1. Откуда взять файлы miniapp

В оригинальном репо есть **app-config.json** в корне; статика miniapp (`index.html` и при необходимости `app-config.json` в папке) может быть в репозитории или выкладываться в `/var/www/remnawave-miniapp` (как в README).

**Вариант A — из оригинального репо (рекомендуется):**

На сервере в любой папке:

```bash
git clone --depth 1 https://github.com/BEDOLAGA-DEV/remnawave-bedolaga-telegram-bot.git bedolaga-origin
cd bedolaga-origin
```

- Если есть папка **miniapp/** — скопируй её в проект бота:  
  `cp -r miniapp /путь/к/ghostvpnbot/`
- Файл **app-config.json** в корне репо — скопируй в папку miniapp или в корень проекта:  
  `cp app-config.json /путь/к/ghostvpnbot/miniapp/` (или оставь в корне, бот ищет оба варианта).

После копирования можно удалить клон: `cd .. && rm -rf bedolaga-origin`.

**Вариант B — статика в отдельной директории (как в README):**

Создай каталог и положи туда файлы miniapp (из репо или собранный фронт):

```bash
sudo mkdir -p /var/www/remnawave-miniapp
# скопируй сюда index.html и при необходимости app-config.json
sudo chown -R www-data:www-data /var/www/remnawave-miniapp
```

В конфиге nginx ниже для bot.ghostvpn.cc используй тогда `root /var/www/remnawave-miniapp;` вместо `root /путь/к/miniapp;`.

---

## 2. Переменные окружения (.env)

В `.env` на сервере добавь или проверь (по [miniapp-setup.md](miniapp-setup.md), раздел 2):

```env
WEB_API_ENABLED=true
WEB_API_HOST=0.0.0.0
WEB_API_PORT=8080
WEB_API_ALLOWED_ORIGINS=https://bot.ghostvpn.cc
WEB_API_DEFAULT_TOKEN=<сгенерируй: openssl rand -hex 32>
```

Режим кнопки «Подключиться» в боте (из README / конфиг бота):

```env
CONNECT_BUTTON_MODE=miniapp_subscription
```

Один раз сгенерируй токен:

```bash
openssl rand -hex 32
```

Подставь полученное значение в `WEB_API_DEFAULT_TOKEN`. Перезапусти бота после правок `.env`.

---

## 3. Nginx для bot.ghostvpn.cc

По [docs/miniapp-setup.md](miniapp-setup.md) (раздел 7) и README — конфиг для **одного домена** bot.ghostvpn.cc (статика + прокси `/miniapp/` на бота).

Создай конфиг (например `/etc/nginx/sites-available/bot.ghostvpn.cc`):

```nginx
server {
    listen 80;
    server_name bot.ghostvpn.cc;
    # редирект на HTTPS после получения сертификата (см. шаг 4)
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name bot.ghostvpn.cc;

    ssl_certificate     /etc/letsencrypt/live/bot.ghostvpn.cc/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/bot.ghostvpn.cc/privkey.pem;

    # Статика miniapp (путь — где у тебя лежит miniapp на сервере)
    root /var/www/remnawave-miniapp;
    index index.html;

    location = /app-config.json {
        add_header Access-Control-Allow-Origin "*";
        try_files $uri =404;
    }

    location / {
        try_files $uri /index.html =404;
    }

    # Проксирование API miniapp на бота (порт 8080 — как в docker-compose)
    location /miniapp/ {
        proxy_pass http://127.0.0.1:8080/miniapp/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Если статика лежит в папке проекта (например `/home/user/ghostvpnbot/miniapp`), замени:

- `root /var/www/remnawave-miniapp;` → `root /home/user/ghostvpnbot/miniapp;`

Если бот в Docker на том же сервере и слушает не localhost, а контейнер, замени `127.0.0.1:8080` на `имя_контейнера_бота:8080` (как в README: `remnawave_bot:8080` при общей сети).

Подключи сайт и проверь конфиг:

```bash
sudo ln -sf /etc/nginx/sites-available/bot.ghostvpn.cc /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## 4. SSL (Let's Encrypt)

По README (раздел «Быстрая настройка SSL для Nginx»): порт 80 должен быть свободен на время выдачи сертификата.

```bash
sudo apt install certbot -y
# если nginx уже слушает 80 — временно останови: sudo systemctl stop nginx
sudo certbot certonly --standalone -d bot.ghostvpn.cc --agree-tos --email твой@email.com --non-interactive
# снова запусти nginx
sudo systemctl start nginx
```

После этого в конфиге nginx уже должны подхватиться пути к сертификатам (как в блоке выше). Если использовал другой способ (например certbot для nginx), пути могут быть те же.

---

## 5. Проверка API

После запуска бота (см. [DEPLOY_SERVER_AND_GIT.md](DEPLOY_SERVER_AND_GIT.md)):

```bash
curl -H "X-API-Key: ТВОЙ_WEB_API_DEFAULT_TOKEN" https://bot.ghostvpn.cc/miniapp/health
# или напрямую к боту:
curl -H "X-API-Key: ТВОЙ_WEB_API_DEFAULT_TOKEN" http://127.0.0.1:8080/health
```

Ожидается ответ с данными о состоянии сервиса (не 401/404).

---

## 6. Кнопка в Telegram (из инструкций в гите)

По [miniapp-setup.md](miniapp-setup.md), раздел 6:

1. В **админке бота**: Конфигурации бота → Прочее → Miniapp — настрой URL miniapp: `https://bot.ghostvpn.cc`.
2. Перезапусти бота.
3. При необходимости в **@BotFather**: `/setmenu` → укажи Web App URL: `https://bot.ghostvpn.cc`.

---

## 7. Проверка работы

1. Открой в браузере: `https://bot.ghostvpn.cc`.
2. Открой miniapp из бота (кнопка «Подключиться» или пункт меню Web App).
3. В DevTools (F12) → Network проверь, что запрос к `https://bot.ghostvpn.cc/miniapp/subscription` возвращает 200 и JSON (а не 401/404).

Если видишь 401 — проверь `WEB_API_ALLOWED_ORIGINS` и токен. Если 404 — проверь, что nginx проксирует `/miniapp/` на бота и бот запущен с `WEB_API_ENABLED=true`. Остальное — по таблице диагностики в [miniapp-setup.md](miniapp-setup.md), раздел 10.

---

## Краткий чеклист

| Шаг | Действие |
|-----|----------|
| 1 | Взять `miniapp/` и/или `app-config.json` из [BEDOLAGA-DEV/remnawave-bedolaga-telegram-bot](https://github.com/BEDOLAGA-DEV/remnawave-bedolaga-telegram-bot) или положить статику в `/var/www/remnawave-miniapp` |
| 2 | В `.env`: `WEB_API_ENABLED=true`, `WEB_API_ALLOWED_ORIGINS=https://bot.ghostvpn.cc`, `WEB_API_DEFAULT_TOKEN=...`, `CONNECT_BUTTON_MODE=miniapp_subscription` |
| 3 | Nginx: статика с `root` на папку miniapp, `location /miniapp/` → proxy на бота :8080 |
| 4 | SSL: `certbot certonly --standalone -d bot.ghostvpn.cc` |
| 5 | Проверка: `curl -H "X-API-Key: ..." https://bot.ghostvpn.cc/miniapp/health` |
| 6 | Админка бота + при необходимости @BotFather: URL miniapp `https://bot.ghostvpn.cc` |

Вся логика и форматы запросов/ответов — как в оригинальном коде и в [docs/miniapp-setup.md](miniapp-setup.md).
