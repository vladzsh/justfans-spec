# Tasks: add-chatter-crm

Контракты (payload, события, схема БД) — в `design.md`; критерии приёмки — в `specs/*/spec.md`. Каждая задача — отдельный коммит (или несколько), формат: `type: short description` (Conventional Commits). Не пушить в main мимо коммитов по ходу работы — жюри смотрит историю.

## 1. Backend: каркас

- [x] 1.1 Инициализировать Django-проект в `backend/` (`justfans-backend`): `config` + приложения `accounts`, `chat`, `monitor`; настройки через env (django-environ или os.environ), PostgreSQL, Redis channel layer, DRF (SessionAuthentication), Channels + Daphne в `asgi.py`
- [x] 1.2 Кастомная модель `accounts.User` с `role` (chatter/teamlead) и `display_name`; `AUTH_USER_MODEL`; миграция
- [x] 1.3 Модели `ContentModel`, `Fan`, `Conversation`, `Message` по ER-схеме design.md: constraints (unique fan+model, unique conversation+client_msg_id, check ppv_price), индекс (conversation_id, id); миграции
- [x] 1.4 Настроить pytest (pytest-django, pytest-asyncio) и заготовку фабрик/фикстур

## 2. Backend: auth и REST

- [x] 2.1 Эндпоинты `POST /api/auth/login/`, `POST /api/auth/logout/`, `GET /api/auth/me/`; пермишены по ролям (IsChatter / IsTeamlead) — спека `auth`
- [x] 2.2 `GET /api/config/` с порогами из env (`OVERDUE_SECONDS`, `PRESENCE_GRACE_SECONDS`, `HEARTBEAT_SECONDS`)
- [x] 2.3 `GET /api/conversations/` — список диалогов чатера с payload из design.md, сортировка по `last_message_at DESC`
- [x] 2.4 `GET /api/conversations/{id}/messages/` — cursor-пагинация `before_id`/`limit`, `has_more`; 404 на чужой диалог — спека `chat`
- [x] 2.5 `POST /api/conversations/{id}/read/` — обнуление `unread_count` (select_for_update) + `conversation.read` в группу чатера через on_commit
- [x] 2.6 `GET /api/sync/?after_id=` — сообщения `id > after_id` по всем диалогам чатера (ASC, cap 500) + снапшоты диалогов — спека `realtime`

## 3. Backend: доменные сервисы и WS

- [x] 3.1 Сервис создания сообщения (общий для чатера и фана): `transaction.atomic` + `select_for_update` диалога, обновление `last_message_at` / `unread_count` / `awaiting_reply_since` по правилам спеки `chat`, идемпотентность по `client_msg_id`, рассылка `message.new` + `monitor.update` через `transaction.on_commit`
- [x] 3.2 Consumer `/ws/`: auth по сессии (4401 без неё), подписка chatter → `chatter.{id}`, teamlead → `monitor`; обработка `message.send` (валидация: непустой text, ppv_price для ppv, свой диалог → иначе `error`) и `ping`/`pong`
- [x] 3.3 Presence: запись `last_seen` в Redis (SETEX, TTL = grace) на connect и каждый ping; `monitor.update` в группу `monitor` на connect/disconnect — спека `teamlead-monitor`
- [x] 3.4 `GET /api/monitor/snapshot/` — чатеры с `connected`, `last_seen`, `dialogs_count`, `waiting[]`
- [x] 3.5 `POST /api/demo/fan-message/` — эмуляция фана (случайный/указанный диалог, случайная фраза) через сервис из 3.1

## 4. Backend: сидер и тесты

- [x] 4.1 `manage.py seed`: 2–3 модели, 3–4 чатера (`chatter1`/`chatter2`/... + общий пароль), 1 тимлид, диалоги с историей (текст + PPV), часть с активным `awaiting_reply_since` и непрочитанными; идемпотентность повторного запуска
- [x] 4.2 Тесты метрики ожидания: фан пишет → таймер ставится один раз; чатер отвечает → сбрасывается; unread инкремент/сброс
- [x] 4.3 Тесты resync: `/api/sync/` возвращает ровно пропущенное (без дублей и потерь), пагинация `before_id` без пересечений
- [x] 4.4 Тесты идемпотентности отправки: повтор `client_msg_id` не создаёт дубль; тест «события не уходят при откате транзакции» (on_commit)
- [x] 4.5 Тест WS-флоу через Channels `WebsocketCommunicator`: connect → message.send → message.new обеим группам; 4401 без сессии

## 5. Frontend: каркас и auth

- [x] 5.1 Инициализировать Vite + Vue 3 (Composition API) + Pinia + Vue Router + Vitest в `frontend/` (`justfans-frontend`); dev-proxy `/api` и `/ws` на backend
- [x] 5.2 Стор `auth` + страница логина; восстановление сессии через `/api/auth/me/`; маршрутизация по роли (chatter → /chat, teamlead → /monitor), guards
- [x] 5.3 WS-сервис: singleton-подключение к `/ws/`, экспоненциальный backoff с jitter, heartbeat `ping` по `heartbeat_seconds`, индикатор состояния соединения, диспетчеризация событий в сторы

## 6. Frontend: рабочее место чатера

- [x] 6.1 Стор `conversations` + список диалогов: аватары, превью последнего сообщения, время, бейдж непрочитанных, индикатор ожидания фана; live-обновление от `message.new`
- [x] 6.2 Стор `messages` + окно переписки: история с подгрузкой старых при скролле вверх (`before_id`, сохранение позиции скролла), стили fan/chatter, отметка PPV с ценой
- [x] 6.3 Отправка: обычное и PPV (поле цены), оптимистичное отображение с `client_msg_id`, подтверждение по `message.new`, ресенд после reconnect
- [x] 6.4 Resync: после reconnect — `GET /api/sync/?after_id=`, merge с дедупликацией по `message.id`, перезапись снапшотов диалогов
- [x] 6.5 Mark-read: при открытии диалога и при `message.new` в открытый диалог; обработка `conversation.read` (вкладки)
- [x] 6.6 Кнопка «симулировать сообщение фана» в открытом диалоге (вызов `/api/demo/fan-message/`)

## 7. Frontend: монитор тимлида

- [x] 7.1 Стор `monitor`: снапшот при входе и после reconnect, применение `monitor.update`
- [x] 7.2 Дашборд: чатеры с online/offline (вычисление по `last_seen` + grace локальным тикером), число диалогов, список ожидающих с таймером ожидания
- [x] 7.3 Подсветка просрочки `> overdue_seconds` локальным тикером + счётчик просрочек по чатеру — спека `teamlead-monitor`

## 8. Frontend: тесты

- [x] 8.1 Vitest: стор `messages` — дедуп по id при merge resync, упорядочивание, merge страницы пагинации с live-сообщениями
- [x] 8.2 Vitest: стор `conversations` — инкремент/сброс непрочитанных, обновление превью; стор `monitor` — вычисление offline и просрочки по timestamp + порогу

## 9. Инфраструктура и сдача

- [x] 9.1 Dockerfile backend (Daphne) и frontend (multi-stage: build SPA → nginx со статикой и прокси `/api`, `/ws`)
- [x] 9.2 `docker-compose.yml` в `justfans-backend`: db, redis, backend (миграции на старте), frontend (build из git URL `justfans-frontend`); `.env.example` с порогами
- [x] 9.3 README backend: запуск, порты, учётки из сидера, эмуляция фана (кнопка + curl), env-пороги, ссылка на деплой и на репо спецификации; README frontend: dev-запуск
- [ ] 9.4 Деплой на Railway: PostgreSQL, Redis, backend (release: migrate + seed), frontend-nginx публичный с прокси по private network; проверить вход и realtime на стенде; ссылку — в README
