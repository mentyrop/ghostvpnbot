# Кассы (платёжные системы): поддержка и условия подключения

Проект основан на [BEDOLAGA-DEV/remnawave-bedolaga-telegram-bot](https://github.com/BEDOLAGA-DEV/remnawave-bedolaga-telegram-bot). Список платёжных систем и способ подключения в оригинале и в этом репозитории **совпадают**.

---

## Какие кассы поддерживает оригинальный гит (и этот проект)

| Касса | Описание | Карты | СБП | Крипто | Регистрация у провайдера |
|-------|----------|-------|-----|--------|---------------------------|
| **YooKassa** | ЮKassa (Яндекс) | ✅ | ✅ (отдельно) | — | [yookassa.ru](https://yookassa.ru) |
| **CloudPayments** | Карты + СБП | ✅ | ✅ | — | [cloudpayments.ru](https://cloudpayments.ru) |
| **Freekassa** | Агрегатор (СБП, карты и др.) | ✅ | ✅ (в т.ч. NSPK) | — | [freekassa.ru](https://freekassa.ru) |
| **KassaAI** | api.fk.life (СБП, карты, SberPay) | ✅ | ✅ | — | Отдельный сервис (fk.life) |
| **Robokassa** | Карты, СБП и др. | ✅ | ✅ | — | [robokassa.ru](https://robokassa.ru) |
| **MulenPay** | Mulen Pay | ✅ | — | — | [mulenpay.ru](https://mulenpay.ru) |
| **Pal24 / PayPalych** | СБП + карты | ✅ | ✅ | — | [pal24.pro](https://pal24.pro) |
| **Platega** | Карты + СБП | ✅ | ✅ | — | [platega.io](https://platega.io) |
| **WATA** | Wata (карты, СБП) | ✅ | ✅ | — | [wata.pro](https://wata.pro) |
| **Tribute** | Банковские карты | ✅ | — | — | [tribute.app](https://tribute.app) |
| **CryptoBot** | Криптовалюта в Telegram | — | — | ✅ | Через @CryptoBot |
| **Heleket** | Криптоплатежи | — | — | ✅ | [heleket.com](https://heleket.com) |
| **Telegram Stars** | Внутренняя валюта Telegram | — | — | звёзды | Не требует отдельной кассы |

---

## Сравнение условий (что нужно для подключения в боте)

Для работы любой кассы в боте нужно:

1. **Зарегистрироваться у провайдера** (ИП/ООО/самозанятый — по правилам выбранной кассы).
2. **Получить ключи/ID** в личном кабинете кассы.
3. **Прописать переменные в `.env`** (см. `.env.example`).
4. **Открыть webhook по HTTPS** и указать URL в кассе (см. ниже).

### Минимальный набор переменных по кассам

| Касса | Включение | Обязательные переменные | Webhook path |
|-------|-----------|-------------------------|--------------|
| **YooKassa** | `YOOKASSA_ENABLED=true` | `YOOKASSA_SHOP_ID`, `YOOKASSA_SECRET_KEY`, `YOOKASSA_RETURN_URL` | `/yookassa-webhook` |
| **CloudPayments** | `CLOUDPAYMENTS_ENABLED=true` | `CLOUDPAYMENTS_PUBLIC_ID`, `CLOUDPAYMENTS_API_SECRET` | `/cloudpayments-webhook` |
| **Freekassa** | `FREEKASSA_ENABLED=true` | `FREEKASSA_SHOP_ID`, `FREEKASSA_API_KEY`, `FREEKASSA_SECRET_WORD_1`, `FREEKASSA_SECRET_WORD_2` | `/freekassa-webhook` |
| **KassaAI** | `KASSA_AI_ENABLED=true` | `KASSA_AI_SHOP_ID`, `KASSA_AI_API_KEY`, `KASSA_AI_SECRET_WORD_2` | `/kassa-ai-webhook` |
| **Robokassa** | `ROBOKASSA_ENABLED=true` | `ROBOKASSA_MERCHANT_LOGIN`, `ROBOKASSA_PASSWORD_1`, `ROBOKASSA_PASSWORD_2` | `/robokassa-webhook` (Result URL). Опционально: чек (Receipt) — см. [фискализация](https://docs.robokassa.ru/ru/fiscalization). |
| **MulenPay** | `MULENPAY_ENABLED=true` | `MULENPAY_API_KEY`, `MULENPAY_SECRET_KEY`, `MULENPAY_SHOP_ID` | `/mulenpay-webhook` |
| **Pal24 (PayPalych)** | `PAL24_ENABLED=true` | `PAL24_API_TOKEN`, `PAL24_SHOP_ID`, `PAL24_SIGNATURE_TOKEN` | `/pal24-webhook` |
| **Platega** | `PLATEGA_ENABLED=true` | `PLATEGA_MERCHANT_ID`, `PLATEGA_SECRET` | `/platega-webhook` |
| **WATA** | `WATA_ENABLED=true` | `WATA_ACCESS_TOKEN`, `WATA_TERMINAL_PUBLIC_ID` | `/wata-webhook` |
| **Tribute** | `TRIBUTE_ENABLED=true` | `TRIBUTE_API_KEY` | `/tribute-webhook` |
| **CryptoBot** | `CRYPTOBOT_ENABLED=true` | `CRYPTOBOT_API_TOKEN`, `CRYPTOBOT_WEBHOOK_SECRET` | `/cryptobot-webhook` |
| **Heleket** | `HELEKET_ENABLED=true` | `HELEKET_MERCHANT_ID`, `HELEKET_API_KEY` | `/heleket-webhook` |

Подробные переменные (лимиты, описания чеков, тестовый режим и т.д.) — в **`.env.example`**, блок «ПЛАТЕЖНЫЕ СИСТЕМЫ».

---

## Что указать в личном кабинете кассы (webhook)

После деплоя бота с HTTPS (например, `https://bot.ghostvpn.cc`) в настройках уведомлений/вебхуков в кассе указываешь **полный URL**:

- ЮKassa: `https://bot.ghostvpn.cc/yookassa-webhook`
- CloudPayments: `https://bot.ghostvpn.cc/cloudpayments-webhook`
- Freekassa: `https://bot.ghostvpn.cc/freekassa-webhook`
- KassaAI: `https://bot.ghostvpn.cc/kassa-ai-webhook`
- Robokassa: в настройках магазина укажи **Result URL** (уведомление об оплате): `https://bot.ghostvpn.cc/robokassa-webhook`
- и т.д. по таблице выше.

Не использовать `http://` или `http://IP:8080/...`. Часть касс не поддерживает нестандартный порт (например, `:8443`) — тогда нужен общий nginx на 443 (см. `docs/DEPLOY_FULL_STEPS.md`, шаги 3.4.9–3.4.11).

### Robokassa: передача данных в чек (ФЗ-54)

Интеграция сделана по [официальной документации](https://docs.robokassa.ru/ru/quick-start): [интерфейс оплаты](https://docs.robokassa.ru/ru/pay-interface), [уведомления](https://docs.robokassa.ru/ru/notifications-and-redirects), [фискализация](https://docs.robokassa.ru/ru/fiscalization). Чтобы данные попадали в чек:

1. Включи: `ROBOKASSA_RECEIPT_ENABLED=true`.
2. При необходимости задай в `.env`: `ROBOKASSA_RECEIPT_SNO` (СНО: osn, usn_income и т.д.), `ROBOKASSA_RECEIPT_TAX` (none, vat20, vat0…), `ROBOKASSA_RECEIPT_PAYMENT_METHOD` (full_payment и т.д.), `ROBOKASSA_RECEIPT_PAYMENT_OBJECT` (service, commodity…), `ROBOKASSA_RECEIPT_ITEM_NAME` (наименование позиции, до 128 символов; пусто = из `PAYMENT_BALANCE_DESCRIPTION`).

При включённом чеке подпись запроса: `MerchantLogin:OutSum:InvId:Receipt:Пароль#1`. Ответ на Result URL: `OK{InvId}`. Опционально проверка IP из `ROBOKASSA_TRUSTED_IPS`.

---

## Где смотреть условия регистрации (комиссии, требования)

Условия приёма (ИП/ООО/самозанятый, комиссии, ограничения по тематике) задаются **провайдером**, не кодом бота. Их нужно смотреть на сайтах касс:

- **YooKassa**: [yookassa.ru](https://yookassa.ru) — раздел для продавцов/подключение.
- **CloudPayments**: [cloudpayments.ru](https://cloudpayments.ru).
- **Freekassa / KassaAI**: сайты агрегаторов (freekassa.ru, fk.life и т.п.).
- **MulenPay**: [mulenpay.ru](https://mulenpay.ru) — есть ограничения по тематике (запрет казино, ставок и т.д., см. `MULENPAY_DESCRIPTION` в `.env.example`).
- **Tribute**: [tribute.app](https://tribute.app).
- **CryptoBot**: через бота @CryptoBot в Telegram.
- **Heleket**: [heleket.com](https://heleket.com).

Для **первой регистрации в кассе** обычно проще всего начать с одной из:

- **YooKassa** — привычный вариант для РФ, карты + СБП, есть тестовый режим.
- **Freekassa** или **KassaAI** — агрегаторы, часто быстрее подключение.
- **Telegram Stars** — без отдельной юр. регистрации в платёжной системе, включается через `TELEGRAM_STARS_ENABLED=true` и настройки бота.

Итог: набор касс в оригинальном гите и в ghostvpnbot **одинаковый**; отличий по условиям подключения в коде нет. Разница только в том, где и как ты регистрируешься у каждого провайдера и какой URL webhook указываешь в кассе.
