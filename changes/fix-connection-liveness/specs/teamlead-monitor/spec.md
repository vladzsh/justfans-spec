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

#### Scenario: Живой чатер не уходит в ложный оффлайн
- **WHEN** чатер онлайн дольше `presence_grace_seconds` без отправки сообщений, но heartbeat идёт
- **THEN** `presence.update` освежает `last_seen`, и чатер остаётся онлайн
