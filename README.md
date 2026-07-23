[README.md](https://github.com/user-attachments/files/30307981/README.md)
# Триединство — мини-диагностика для Telegram

Telegram-приложение (бот + Mini App), которое проводит короткую диагностику
из 15 вопросов по трём опорам жизни — **Тело**, **Разум**, **Дух** — и выдаёт
персональный архетип с конкретными прикладными советами по каждой сфере.

Пройти тест можно двумя способами:

- **В чате с ботом** — пошагово, кнопками 1–5 под каждым вопросом.
- **В Mini App** — красивый веб-интерфейс внутри Telegram с прогресс-баром,
  анимированными шкалами результата и разворачивающимися блоками советов.

## Структура монорепозитория

```
packages/
  shared/   — вопросы, подсчёт баллов, архетипы, база советов (используется и ботом, и сервером)
  server/   — Express API: приём ответов, подсчёт результата, выдача Mini App
  bot/      — Telegram-бот на grammY (long polling)
  webapp/   — Mini App: Vite + React + TypeScript + Tailwind
```

## Быстрый старт (локально)

1. Установите зависимости из корня:

   ```bash
   npm install
   ```

2. Создайте бота через [@BotFather](https://t.me/BotFather) и получите `BOT_TOKEN`.

3. Скопируйте env-файлы и заполните токен:

   ```bash
   cp packages/server/.env.example packages/server/.env
   cp packages/bot/.env.example packages/bot/.env
   cp packages/webapp/.env.example packages/webapp/.env
   ```

   В `packages/server/.env` и `packages/bot/.env` укажите один и тот же `BOT_TOKEN`.

4. Соберите общий пакет и запустите всё сразу (сервер + бот + Mini App в dev-режиме):

   ```bash
   npm run dev
   ```

   - API сервер: http://localhost:3001
   - Mini App (dev): http://localhost:5173
   - Бот стартует в режиме long polling и сразу отвечает на `/start`.

На этом этапе диагностику в чате бота (`💬 Пройти прямо в чате`) уже можно
проходить полностью. Для кнопки **📱 Пройти в мини-приложении** и для открытия
Mini App внутри Telegram нужен публичный HTTPS-адрес (см. ниже) — Telegram не
принимает `http://localhost` как `web_app` URL.

## Подключение Mini App к Telegram (публичный HTTPS)

Telegram требует HTTPS-адрес для `web_app` кнопок. Проще всего для разработки —
пробросить локальный сервер наружу через [ngrok](https://ngrok.com/) (или
Cloudflare Tunnel):

```bash
ngrok http 3001
```

1. Соберите Mini App и запустите сервер в режиме, где он сам раздаёт билд:

   ```bash
   npm run build
   npm run dev:server   # или: node packages/server/dist/index.js после build:server
   ```

   Сервер (`packages/server/src/index.ts`) уже настроен отдавать
   `packages/webapp/dist` по тому же origin, что и API — отдельный домен для
   фронтенда не нужен.

2. Скопируйте HTTPS-адрес из ngrok (например `https://abcd1234.ngrok-free.app`)
   и укажите его в `packages/bot/.env`:

   ```
   WEBAPP_URL=https://abcd1234.ngrok-free.app
   ```

3. Перезапустите бота (`npm run dev:bot`). Кнопка «📱 Пройти в мини-приложении»
   в `/start` теперь откроет Mini App внутри Telegram.

4. (Опционально) Пропишите тот же URL через `@BotFather` → `Bot Settings` →
   `Menu Button`, чтобы Mini App открывалась и через кнопку меню чата.

## Продакшен-деплой (Railway / Render)

Проект — 2 постоянно работающих Node.js-процесса из одного репозитория:
**сервер** (API + отдаёт собранный Mini App, нужен публичный HTTPS-домен) и
**бот** (long polling, публичный домен не нужен, но процесс должен работать
24/7 — это не HTTP-сервис, который может «спать»).

Локальный git-репозиторий уже инициализирован (`git init` + первый коммит).
Дальше:

### 0. Выложить код на GitHub

```bash
# создайте пустой репозиторий на github.com (без README/gitignore), затем:
git remote add origin https://github.com/<ваш-аккаунт>/trinity-app.git
git branch -M main
git push -u origin main
```

### Вариант 1 — Railway (2 сервиса из одного репо)

1. [railway.app](https://railway.app) → залогиниться через GitHub → **New Project → Deploy from GitHub repo** → выбрать `trinity-app`.
2. Railway создаст первый сервис — переименуйте в `trinity-server`, задайте:
   - **Build Command**: `npm install && npm run build`
   - **Start Command**: `node packages/server/dist/index.js`
   - **Variables**: `BOT_TOKEN`, `WEBAPP_ORIGIN=*`, `REQUIRE_TELEGRAM_AUTH=true`
   - **Settings → Networking → Generate Domain** — получите публичный `https://trinity-server-production.up.railway.app`
3. В том же проекте **New → GitHub Repo** (тот же репозиторий) — второй сервис `trinity-bot`:
   - **Build Command**: `npm install && npm run build`
   - **Start Command**: `node packages/bot/dist/bot.js`
   - **Variables**: `BOT_TOKEN` (тот же), `WEBAPP_URL` = домен из шага 2
   - Публичный домен этому сервису не нужен — просто оставьте без Networking.
4. Оба сервиса передеплоятся автоматически при каждом `git push` в `main`.

### Вариант 2 — Render (Blueprint, оба сервиса одной кнопкой)

В репозитории уже есть [`render.yaml`](render.yaml), описывающий оба сервиса.

1. [render.com](https://render.com) → залогиниться через GitHub → **New + → Blueprint** → выбрать репозиторий `trinity-app`. Render сам прочитает `render.yaml` и предложит создать `trinity-server` (web) и `trinity-bot` (worker).
2. После создания — в каждом сервисе зайти в **Environment** и вписать вручную (они намеренно не хранятся в репозитории):
   - `trinity-server`: `BOT_TOKEN`
   - `trinity-bot`: `BOT_TOKEN` (тот же) и `WEBAPP_URL` = публичный URL `trinity-server` (Render выдаёт домен вида `https://trinity-server.onrender.com` сразу после первого деплоя — впишите его сюда и передеплойте `trinity-bot`).
3. `trinity-bot` — тип `worker`, что важно для Render: бесплатный тариф для фоновых воркеров может быть недоступен/ограничен — проверьте актуальные условия на их странице тарифов; для бота, который должен жить 24/7, обычно нужен минимальный платный план.

### После деплоя (оба варианта)

1. В [@BotFather](https://t.me/BotFather) → **Bot Settings → Menu Button** → вписать домен `trinity-server` — тогда Mini App открывается и постоянной кнопкой меню чата.
2. Открыть бота в Telegram, `/start`, проверить оба пути (чат и Mini App).
3. Без своего домена оба хостинга выдают достаточный для Telegram публичный поддомен (`*.up.railway.app` / `*.onrender.com`) — свой домен не обязателен, можно подключить позже через **Settings → Custom Domain**.

### Важно про хранилище

Данные результатов диагностики по умолчанию хранятся в `data/results.json`
(простое JSON-хранилище без внешних зависимостей, живёт на диске сервиса).
На Railway/Render это подходит для MVP, но при передеплое или нескольких
инстансах сервиса файл может не сохраниться между деплоями — для реальной
нагрузки замените `packages/server/src/storage.ts` на настоящую БД (сигнатуры
`saveResult`/`getResult` уже изолируют это место) или подключите постоянный
диск (Railway Volume / Render Disk).

## Как устроена диагностика

- 15 вопросов (`packages/shared/src/questions.ts`), по 5 на каждую из трёх
  опор, шкала согласия 1–5.
- Балл по опоре переводится в проценты 0–100
  (`packages/shared/src/scoring.ts`).
- Для каждой опоры и уровня (низкий/средний/высокий) — свой блок из 4
  конкретных действий (`packages/shared/src/advice.ts`), итого 9 блоков и 36
  прикладных рекомендаций.
- По соотношению трёх баллов определяется **архетип** личности
  (`packages/shared/src/archetypes.ts`) — например, «Атлет без компаса» или
  «Мудрец в уставшем теле» — плюс отдельно выделяется «зона фокуса» (самая
  слабая опора) с точечной рекомендацией, с чего начать.

Вся эта логика находится в `@trinity/shared` и используется одинаково и
ботом, и сервером — расхождений в подсчёте между чатом и Mini App нет.

## Команды

| Команда | Что делает |
| --- | --- |
| `npm run dev` | Сборка shared + параллельный запуск сервера, бота и Mini App (dev) |
| `npm run build` | Полная сборка (shared → webapp → server) для продакшена |
| `npm run dev:server` / `dev:bot` / `dev:webapp` | Запуск только одного сервиса |
