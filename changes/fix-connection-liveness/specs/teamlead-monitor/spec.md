# teamlead-monitor

## MODIFIED Requirements

### Requirement: Presence чатеров
Сервер ДОЛЖЕН отмечать чатера онлайн при открытии WS-соединения и обновлять `last_seen` на каждый heartbeat (Redis, `SETEX` с TTL = `presence_grace_seconds`). При connect и disconnect в группу `monitor` ДОЛЖЕН уходить `monitor.update`. На каждый `ping` чатера сервер ДОЛЖЕН рассылать в группу `monitor` лёгкое событие `presence.update` с `{chatter_id, last_seen}`, чтобы значение `last_seen` у тимлида освежалось между connect/disconnect/сообщениями.

Клиент монитора ДОЛЖЕН вычислять оффлайн по устареванию `last_seen`: чатер оффлайн, когда `now - last_seen > presence_grace_seconds`. Флаг `connected=false` (приходящий со штатным disconnect) — быстрый путь, переводящий чатера в оффлайн немедленно, но НЕ является обязательным условием: при «тихом» обрыве, когда `disconnect` не приходит и heartbeat прекращается, устаревший `last_seen` сам по себе ДОЛЖЕН давать оффлайн.

#### Scenario: Чатер подключился
- **WHEN** чатер открывает WS-соединение
- **THEN** тимлид видит его онлайн без перезагрузки страницы

#### Scenario: Кратковременный обрыв внутри grace period
- **WHEN** соединение чатера оборвалось и восстановилось быстрее `presence_grace_seconds`
- **THEN** на мониторе чатер не «мигает» оффлайном

#### Scenario: Жёсткий обрыв без disconnect
- **WHEN** соединение чатера умерло без события disconnect (обрыв сети) и heartbeat прекратился
- **THEN** не позднее `presence_grace_seconds` монитор показывает чатера оффлайн (по устареванию `last_seen`)

## ADDED Requirements

### Requirement: Рекомендуемые значения пороговых настроек
Дефолтные значения `heartbeat_seconds` и `presence_grace_seconds` ДОЛЖНЫ удовлетворять инварианту `presence_grace_seconds ≥ 3 × heartbeat_seconds`, обеспечивая быстрое обнаружение при сохранении стабильности (без ложных offline-флагов). Рекомендуемые дефолты: `heartbeat_seconds = 5`, `presence_grace_seconds = 15` (коэффициент 3×). При таких значениях монитор показывает offline через ≤ 15 с после тихого обрыва.

#### Scenario: Быстрый переход в offline при снижённых порогах
- **WHEN** `heartbeat_seconds = 5` и `presence_grace_seconds = 15`, чатер пропал без штатного disconnect
- **THEN** `last_seen` устаревает за 15 с; монитор показывает чатера оффлайн не позднее 15 с после последнего успешного ping

#### Scenario: Инвариант стабильности при рекомендованных порогах
- **WHEN** `heartbeat_seconds = 5` и `presence_grace_seconds = 15` (коэффициент = 3)
- **THEN** единственное запоздавшее heartbeat-обновление не приводит к ложному offline; grace (15 с) достаточно, чтобы два последующих heartbeat (суммарно 10 с) успели обновить `last_seen`

#### Scenario: Живой чатер не уходит в ложный оффлайн
- **WHEN** чатер онлайн дольше `presence_grace_seconds` без отправки сообщений, но heartbeat идёт
- **THEN** `presence.update` освежает `last_seen`, и чатер остаётся онлайн
