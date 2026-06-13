# Proposal: fix-connection-liveness

## Why

E2E-тестирование фазы 1 (см. `BUGS.md` в корне) выявило, что система не обнаруживает «тихий» обрыв соединения — разрыв сети без штатного закрытия сокета, при котором ни клиент, ни сервер не получают уведомления. Из-за этого тимлид видит выпавшего чатера «онлайн» бесконечно (BUG-1), а индикатор соединения у чатера не реагирует на пропажу сети (BUG-2). Дополнительно: WS close code 4401 ненаблюдаем клиентом (BUG-3, minor). Это нарушает требование ТЗ о надёжности соединения и ключевую функцию монитора тимлида.

## What Changes

- **Серверный heartbeat-push:** на `ping` от чатера сервер рассылает в группу `monitor` обновление `last_seen` (лёгкое `monitor.update`), чтобы у тимлида значение освежалось не только на connect/disconnect/сообщение.
- **Оффлайн по устареванию `last_seen`:** клиент монитора считает чатера оффлайн, когда `now - last_seen > presence_grace_seconds` — без жёсткого требования `connected=false`. Это устраняет внутреннее противоречие текущей спеки (при тихом обрыве `connected` остаётся `true` навсегда).
- **Клиентский liveness:** WS-клиент SPA отслеживает `pong`; если ответа нет в пределах окна (кратного `heartbeat_seconds`), принудительно закрывает сокет и инициирует reconnect — индикатор и resync срабатывают на тихий обрыв.
- **Наблюдаемый 4401:** consumer выполняет `accept()` перед `close(code=4401)`, чтобы клиент действительно получал код (а не `1006`).
- **Постоянный E2E-сьют** (Playwright) на месте `.explore/`-черновиков, с исправленными дефектами харнесса (CSRF-хелпер, ESM), регрессионными тестами на BUG-1/2/3 и на ранее починенный баг сериализации сообщения фана.

## Capabilities

### New Capabilities

Нет.

### Modified Capabilities

- `realtime`: добавляется требование обнаружения тихого обрыва клиентом (liveness/pong-timeout).
- `teamlead-monitor`: меняется правило вычисления оффлайн (по устареванию `last_seen`) и добавляется серверный heartbeat-push `last_seen` в группу `monitor`.

## Impact

- Backend: `chat/consumers.py` (ping→monitor-push, accept-then-4401), `chat/services.py`/`chat/presence.py` (лёгкий снапшот presence).
- Frontend: `src/services/ws.ts` (pong-timeout), `src/stores/monitor.ts` (оффлайн по `last_seen`).
- Tests: новый постоянный Playwright-сьют в `frontend/` (заменяет `.explore/`), backend-тест на accept-then-4401.
- Контракты: новое лёгкое событие `presence.update` (или переиспользование `monitor.update`) — фиксируется в design.md.
