# Crown Sky — сайт + Telegram-бот + Mini App

Один Web Service на Render отдаёт сайт, Mini App, API и вебхук Telegram. База — Neon.

- `/` — лендинг;
- `/webapp.html` — Mini App;
- `/telegram-webhook` — обновления от Telegram;
- `/api/*` — регистрация, новости, аватарки, записки, статистка;
- `/health` — проверка сервиса.

База данных — **Neon (Serverless Postgres)**: пользователи, залы, места, билеты, новости и записки.

Подробная пошаговая инструкция: **DEPLOY.md**.

## Структура

```
bot/server.mjs      Express + grammY, API и вебхук
bot/db.mjs          Запросы к Neon
bot/telegram.mjs    Проверка initData и утилиты
bot/schema.sql          Таблицы и стартовые данные
site/index.html   Лендинг
site/webapp.html  Mini App
site/config.js    Пустой — API на том же домене, менять не нужно
site/assets/      Картинки лендинга
render.yaml         Конфиг Render
bot/scripts/init-db.mjs Применение схемы к Neon
```

## 1. Neon

1. Создайте проект и базу в https://neon.tech.
2. Скопируйте connection string в `DATABASE_URL`.
3. Выполните:

```bash
cp .env.example .env
# отредактируйте .env
npm install
npm run init-db
```

Скрипт можно запускать повторно: он безопасен и добавит новые колонки/таблицы
(`ticket_no`, `reviews`) в уже существующую базу.

## 2. Telegram-бот

1. Создайте бота у [@BotFather](https://t.me/BotFather), получите `BOT_TOKEN`.
2. Укажите `BOT_USERNAME` (без `@`).
3. После деплоя включите Mini App: `/setmenubutton` → `https://<ваш-сервис>.onrender.com/webapp.html`.
4. При старте сервис сам вызовет `setWebhook` (если `SET_WEBHOOK=true`).
   Проверить можно так:

```bash
npm run webhook:info
```

## 3. Render (один сервис)

Создайте **Web Service** из репозитория (или Blueprint по `render.yaml`):

- Build Command: `npm install`
- Start Command: `npm start`
- Health Check Path: `/health`

Environment Variables: `BOT_TOKEN`, `BOT_USERNAME`, `PUBLIC_URL` (адрес сервиса),
`DATABASE_URL`, `ADMIN_IDS`, `WEBHOOK_SECRET`, `SET_WEBHOOK=true`.

`PUBLIC_SITE_URL` и `SITE_ORIGIN` больше не нужны. `config.js` трогать не нужно —
API определяется как `location.origin`. В `site/index.html` замените только
`BOT_USERNAME` на username своего бота.

Проверьте: `https://<ваш-сервис>.onrender.com/health`.

## Команды бота

| Команда | Что делает |
|---|---|
| `/start` | Приветствие и кнопки приложения |
| `/start seat_12` | Продолжение регистрации с сайта (ссылка уже формируется лендингом) |
| `/news` | Последние новости |
| `/status` | Статус билета и место |
| `/help` | Справка |
| `/news_add Заголовок \| Текст` | Новая новость (только для `ADMIN_IDS`) |

Кнопки: «Продолжить в приложении» (Mini App), «Подтвердить регистрацию», «Новости», «Мой билет».

## Mini App

- Читает `initData` Telegram и `start_param=seat_N`.
- Позволяет ввести имя, сохранить регистрацию и подтвердить её.
- Показывает список новостей из Neon.
- Проверяет подпись `initData` на сервере (HMAC `WebAppData`), поэтому подделать регистрацию нельзя.

## Что сохраняется в Neon

| Данные | Где | Зачем |
|---|---|---|
| Пользователи Telegram | `users`: id, username, first_name, **photo_url**, имя на билете | Аватарки и имена пилотов |
| Залы и места | `halls`, `seats` (по 126 мест в зале), `hall_id` | Бесконечная лента залов: новые залы создаются при скролле |
| Места и билеты | `users.seat_id`, `ticket_no`, `status` | Кто и где сидит, номер билета |
| Ожидающие регистрации | `users.pending_seat`, `status` | Продолжение регистрации из бота/сайта |
| Новости | `news` | Лента в боте и Mini App |
| Записки чата | `reviews`: name, text, photo_url, telegram_id | Чат на сайте теперь хранится в БД, а не только в браузере |

Бесконечные залы отдаёт/создаёт `GET /api/halls?offset=2&limit=1`: сайт подгружает следующий зал при скролле вниз, Mini App — кнопкой «Показать ещё зал». ID места глобальный: `зал N = (N-1)*126 … N*126-1`, поэтому ссылка `?start=seat_N` однозначно указывает и на зал, и на кресло.

Счётчик «ЧЕЛОВЕКА ЖДУТ ИГРУ» берётся из `GET /api/stats` — это число реально подтверждённых пользователей в Neon. Он не зависит от того, сколько залов вы пролистали.

Аватарки отдаёт `GET /api/subscribers` — сайт и Mini App ставят их на занятые кресла.
Записки отдаёт/принимает `GET, POST /api/reviews`. Если API недоступен, сайт использует локальный fallback.

## Важно

- В лендинге константа `BOT_USERNAME` пока равна `crown_sky_bot`. Замените её на username своего бота, чтобы ссылки на бота вели правильно.
- Бесплатный план Render засыпает — первый запрос после простоя может занять несколько секунд.
- Аватарки подписчиков на сайте по-прежнему подключаются через `window.CROWN_SKY_SUBSCRIBERS` или `SUBSCRIBERS_URL`; теперь источником может служить API на базе `users` из Neon.
