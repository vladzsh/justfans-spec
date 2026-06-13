# teamlead-monitor

## MODIFIED Requirements

### Requirement: Снапшот дашборда — агрегация по моделям

`GET /api/monitor/snapshot/` ДОЛЖЕН возвращать ключ `models` рядом с существующим `chatters`. Каждая запись модели ДОЛЖНА содержать: `id`, `name`, `avatar`, `dialogs_count` (общее число диалогов модели), `waiting[]` — диалоги модели, у которых `awaiting_reply_since` не null, в формате `{conversation_id, fan_name, waiting_since}` (ISO 8601). Список ДОЛЖЕН включать все модели (в том числе с нулём ожидающих), упорядоченные по `id`. Просрочка на сервере НЕ вычисляется — клиент считает её сам по `waiting_since` и порогу `overdue_seconds`.

#### Scenario: Снапшот включает все модели
- **WHEN** тимлид запрашивает `GET /api/monitor/snapshot/`
- **THEN** в ответе есть ключ `models` со списком всех контент-моделей, упорядоченных по `id`

#### Scenario: Модель с активными ожиданиями
- **WHEN** у модели есть диалоги с непустым `awaiting_reply_since`
- **THEN** они попадают в `waiting[]` для этой модели с полями `conversation_id`, `fan_name`, `waiting_since`

#### Scenario: Модель без ожидающих диалогов всё равно присутствует
- **WHEN** у модели нет ни одного диалога с `awaiting_reply_since`
- **THEN** модель всё равно есть в `models` с `waiting: []` и корректным `dialogs_count`

#### Scenario: Агрегация поверх всех чатеров
- **WHEN** диалоги модели назначены разным чатерам
- **THEN** `dialogs_count` и `waiting[]` агрегируют данные по всем чатерам, а не только по одному

### Requirement: Форма payload monitor.update

WS-событие `monitor.update` ДОЛЖНО доставлять payload в форме `{"chatter": <ChatterStatus>, "models": [<ModelStatus>, ...]}`. Поле `chatter` — тот же объект, что раньше передавался как корень payload. Поле `models` — полный список всех моделей (тот же формат, что в REST-снапшоте). Каждый broadcast `monitor.update` (connect, disconnect, send, read) ДОЛЖЕН включать свежеперевычисленный массив `models`.

#### Scenario: monitor.update несёт данные чатера и модели
- **WHEN** чатер отправляет сообщение
- **THEN** тимлид получает `monitor.update` с полями `payload.chatter.id` и `payload.models` (непустой список)

#### Scenario: monitor.update при connect/disconnect также несёт models
- **WHEN** чатер подключается или отключается
- **THEN** `monitor.update` содержит как `chatter`, так и `models`
