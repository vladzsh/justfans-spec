# Tasks: add-monitor-models-panel

Контракты — в `specs/teamlead-monitor/spec.md`; форма payload — там же. Каждая задача — отдельный коммит, Conventional Commits.

## 1. Backend

- [x] 1.1 Добавить `build_models_snapshot()` в `chat/services.py`: агрегация по `ContentModel`, `dialogs_count`, `waiting[]` с `conversation_id` / `fan_name` / `waiting_since` (ISO 8601), сортировка по `id`
- [x] 1.2 `monitor/views.py` — добавить `"models": build_models_snapshot()` в ответ `GET /api/monitor/snapshot/`
- [x] 1.3 `chat/consumers.py` — изменить payload всех `group_send("monitor", ...)` на `{"chatter": chatter_snapshot, "models": models_snapshot}` (3 места: `_push_monitor_update`, `_handle_message_send`, `broadcast_message_new_sync`)
- [x] 1.4 Тесты `build_models_snapshot()`: модель без диалогов (пустой `waiting`, `dialogs_count=0`), модель с ожидающим диалогом, агрегация поверх нескольких чатеров, нет ожидающих после ответа чатера, порядок по `id`
- [x] 1.5 Тест REST-снапшота: ключ `models` присутствует, модель без ожидающих всё равно в списке
- [x] 1.6 Тест WS: `monitor.update` несёт `payload.chatter.id` и `payload.models`

## 2. Frontend

- [x] 2.1 Обновить стор `monitor` — применение `monitor.update`: читать `event.chatter` вместо корня payload; добавить поле `models` в состояние
- [x] 2.2 Обновить REST-снапшот (`GET /api/monitor/snapshot/`): сохранять `models` в стор при загрузке и после reconnect
- [x] 2.3 Компонент панели моделей: таблица/список моделей с `name`, `avatar`, `dialogs_count`, `waiting.length`; подсветка просрочки по `waiting_since` + `overdue_seconds` локальным тикером
- [x] 2.4 Интеграция панели моделей в страницу `/monitor` рядом с панелью чатеров
- [x] 2.5 Vitest: стор `monitor` — применение `monitor.update` с новой формой payload; модель с нулём ожидающих отображается; модель с ожидающими попадает в `waiting`
