# Tasks: fix-connection-liveness

Контракты — в `design.md`; критерии приёмки — в дельтах `specs/realtime`, `specs/teamlead-monitor` и в существующих спеках. Стек поднимается `cd backend && FRONTEND_CONTEXT=../frontend docker compose up --build` (:8080). Каждая задача — отдельный коммит (Conventional Commits, английский), в соответствующем репо.

## 1. Постоянный E2E-сьют (инфраструктура)

- [ ] 1.1 Перенести черновики из `frontend/.explore/` в постоянный `frontend/tests/e2e/`; настроить `playwright.config.ts` (baseURL :8080, list-reporter, скриншоты при падении). Удалить `.explore/` из активного использования
- [ ] 1.2 Хелпер авторизованного API-запроса: логин → извлечь `csrftoken` из cookie → слать `X-CSRFToken` + `Referer`. Заменить голые `request`-вызовы — снять ложные 403 (бывшие DEMO-3, CHAT-9, CHAT-10). Переписать ESM-нарушение (`require`→`import`) из DEMO-5
- [ ] 1.3 Разнести быстрые и медленные (overdue/grace) тесты на два Playwright-проекта; для медленных — отдельный compose-стек со сниженными `OVERDUE_SECONDS`/`PRESENCE_GRACE_SECONDS` через env, чтобы не ждать реальные 30с

## 2. BUG-2 — клиентский liveness (realtime)

- [ ] 2.1 `frontend/src/services/ws.ts`: учёт `pong` — таймер на каждый `ping`, сброс по `pong`; при таймауте (`≈2 × heartbeat_seconds`) `socket.close()` → существующий reconnect/resync. Индикатор переходит в «переподключение»
- [ ] 2.2 Vitest: pong пришёл → сокет жив; pong не пришёл за окно → принудительный close и переход в reconnect (с фейковыми таймерами)
- [ ] 2.3 E2E (медленный проект): `setOffline(true)` → индикатор уходит из «онлайн» в пределах окна; `setOffline(false)` → возврат в «онлайн» + resync без дублей

## 3. BUG-1 — серверный heartbeat-push + оффлайн по last_seen (teamlead-monitor)

- [ ] 3.1 Backend `chat/consumers.py`: на `ping` чатера рассылать в группу `monitor` событие `presence.update {chatter_id, last_seen}` (лёгкий payload, без полного снапшота)
- [ ] 3.2 Frontend `src/stores/monitor.ts`: обрабатывать `presence.update` (освежать только `last_seen`); изменить `calcIsOffline` — оффлайн при `now - last_seen > grace` независимо от `connected` (а `connected=false` остаётся быстрым путём)
- [ ] 3.3 Vitest `src/stores/monitor.ts`: живой чатер с идущим heartbeat не уходит в оффлайн после grace; чатер без heartbeat (замёрзший `last_seen`) уходит в оффлайн после grace даже при `connected=true`
- [ ] 3.4 E2E (медленный проект): тимлид + чатер; `setOffline` чатера → не позднее grace монитор показывает «Офлайн» (регрессия MON-9)

## 4. BUG-3 — наблюдаемый 4401 (auth)

- [ ] 4.1 Backend `chat/consumers.py`: для неавторизованных `accept()` затем `close(code=4401)` вместо close до accept
- [ ] 4.2 E2E/тест: WS без сессии → клиент получает close code `4401` (а не `1006`)

## 5. Регрессия и зелёный прогон

- [ ] 5.1 E2E: сообщение фана в УЖЕ открытый диалог → появляется в переписке и unread сбрасывается (регрессия ранее починенного бага сериализации `c2e81ea`)
- [ ] 5.2 Прогнать весь постоянный сьют против Docker-стека — зелёный; обновить `BUGS.md` (отметить закрытые BUG-1/2/3) или заменить кратким E2E-README
