---
version: "1.0.0"
updated: "2026-05-24"
agent: "architect"
status: draft
generated-from:
  - template: "templates-feat/180-tech-flow/tech-flow-template.md"
    feature: "tech-flow"
    agent: "architect"
    at: "2026-05-24"
    patch: false
---

# Technical Flow: UpdateParty

**Entity:** party
**Связанные документы:** Use Case (uc-party-UpdateParty), Domain Model

---

## Входные данные

Операция запускается асинхронно по получению события обновления профиля организации от IdS (через брокер сообщений). PATCH-семантика — payload содержит только изменяемые DynamicAttributes.

**DTO:** PartyProfileUpdatedEvent (из IdS)

| Поле | Тип | Обязательное | Описание |
|------|-----|-------------|----------|
| PartyId | UUID (string) | Да | Organization ID из IdS. Идентификатор организации, чей кэш обновляется |
| DynamicAttributes | jsonb | Нет | Динамический набор полей для слияния. PATCH-семантика: переданные ключи перезаписываются, отсутствующие — не изменяются |
| IdempotencyKey | UUID | Да | Ключ идемпотентности для защиты от дубликатов. Является `version_uuid` после записи (см. db-patterns) |

**Контекст вызова:**
- CorrelationId — сквозной идентификатор запроса (из сообщения брокера)
- IdempotencyKey — ключ идемпотентности для deduplication. Генерируется IdS как UUID v4
- SourceSystem — IdS (внешняя система-источник)
- Timestamp — метка времени события из IdS (используется для HLC merge)

---

## Порядок выполнения (Workers)

Цепочка Workers в порядке вызова. Операция выполняется синхронно внутри обработчика события до записи в outbox.

| # | Worker | Действие | Вход | Выход |
|---|--------|----------|------|-------|
| 1 | IdempotencyValidator | Проверить idempotency-ключ события по уникальности (entity_id, idempotency_key); если ключ уже обработан — завершить с пропуском (IDEMPOTENCY_DUPLICATE) | PartyProfileUpdatedEvent (PartyId, IdempotencyKey) | void (или skip) |
| 2 | PartyLoader | Загрузить актуальную версию Party из entity_ledger по PartyId (максимальная по HLC total order). Если Party не найдена — ошибка NOT_FOUND | PartyProfileUpdatedEvent (PartyId) | Party aggregate (текущая версия + version_uuid) |
| 3 | PayloadValidator | Проверить, что DynamicAttributes в payload не пустой (содержит хотя бы один ключ для обновления). Проверить отсутствие Mandatory-полей (CacheStatus, Inn, PartyName, RegistrationType, PartyId) в DynamicAttributes — при наличии залогировать предупреждение, но не блокировать | PartyProfileUpdatedEvent (DynamicAttributes) + Party aggregate | void (или warning) |
| 4 | DynamicAttributesMerger | Выполнить PATCH-merge: значения из payload.DynamicAttributes перезаписывают соответствующие ключи в Party.DynamicAttributes. Ключи, отсутствующие в payload, не изменяются. Обновить updatedAt = текущее время | Party aggregate, PartyProfileUpdatedEvent (DynamicAttributes) | Party aggregate (обновлённый) |
| 5 | HLCGenerator | Сгенерировать 4-компонентный HLC (hlc_l, hlc_c, hlc_instance_id, hlc_dc_id) для новой ledger-записи. version_uuid = idempotency_key | Party aggregate, IdempotencyKey | Party aggregate + HLC (hlc_l, hlc_c, hlc_instance_id, hlc_dc_id) |
| 6 | PartyLedgerWriter | INSERT в entity_ledger (entity_id, version_uuid, hlc_l, hlc_c, hlc_instance_id, hlc_dc_id, origin_dc, idempotency_key, payload) в рамках Append-Only. Только INSERT. Проверка optimistic locking через expected_version_uuid (текущая версия на момент чтения Worker #2). Запись в outbox-таблицу в рамках одной транзакции | Party aggregate (version_uuid, HLC, payload) | void |
| 7 | OutboxWriter | Записать событие PartyCacheUpdated в outbox-таблицу (entity_id, event_type, payload, status=pending). Атомарно с INSERT в entity_ledger | Party aggregate | void |

---

## Транзакционные границы

| Worker (#) | В транзакции? | Комментарий |
|-----------|-------------|-------------|
| 1–5 | Да | Валидация, загрузка, мерж и генерация HLC — в памяти, в рамках одного потока выполнения |
| 6–7 | Да | Единая БД-транзакция: INSERT в entity_ledger (Append-Only) + outbox-запись. При сбое — полный откат. Условие: INSERT только при совпадении expected_version_uuid с актуальной версией |
| 8 (outbox relay) | Нет | Внешняя публикация события — фоновый outbox relay. Outbox-запись гарантирует at-least-once доставку |

---

## Гарантии консистентности

| Гарантия | Обеспечивается |
|----------|---------------|
| Atomicity | Workers 6–7 в одной БД-транзакции: INSERT в entity_ledger + INSERT в outbox. PATCH-мерж (Worker #4) — в памяти, до транзакции |
| Consistency | Инварианты: PartyId существует (Worker #2), payload не пустой (Worker #3). Mandatory-поля не изменяются — PATCH только DynamicAttributes. ИНН, наименование, RegistrationType, CacheStatus — неизменны через этот UC |
| Isolation | Append-Only Ledger: total order через HLC (hlc_l, hlc_c, hlc_instance_id, hlc_dc_id). Optimistic locking через проверку expected_version_uuid = version_uuid актуальной записи (Worker #6). Конкурентные записи с разными idempotency_key — упорядочиваются по HLC |
| Durability | После commit транзакции: новая версия в entity_ledger (INSERT), outbox-запись сохранена. Событие будет доставлено outbox relay. Полная история изменений благодаря Append-Only |
| Exactly-once обработка | IdempotencyKey ≡ version_uuid (один ключ — одна версия). Unique constraint (entity_id, idempotency_key) предотвращает дубликаты. Outbox гарантирует ровно одну публикацию события PartyCacheUpdated |

---

## Обработка ошибок и retry

| Worker (#) | Ошибка | Retry? | Действие |
|-----------|--------|--------|----------|
| 1 | IDEMPOTENCY_DUPLICATE — ключ уже обработан (entity_id, idempotency_key) | Нет | Игнорировать событие. Завершение без ошибки. Логировать IDEMPOTENCY_DUPLICATE |
| 1 | IDEMPOTENCY_KEY_MISMATCH — ключ существует, но payload не совпадает | Нет | Пометить событие как failed → DLQ. Логировать IdempotencyKeyMismatch — возможна replay-атака. Требуется ручное разрешение |
| 2 | NOT_FOUND — Party с указанным PartyId не найдена в entity_ledger | Нет | Пометить событие как failed → DLQ. Логировать PartyNotFound |
| 3 | DynamicAttributes пуст (payload не содержит полей для обновления) | Нет | Логировать EmptyUpdatePayload (предупреждение). Событие считается успешно обработанным — холостое событие от IdS. Завершение без ошибки |
| 3 | Payload содержит Mandatory-поля (CacheStatus, Inn, PartyName, RegistrationType, PartyId) | Нет | Логировать UnexpectedFieldsInPayload (предупреждение). Mandatory-поля игнорируются. DynamicAttributes из payload обновляются. Событие считается успешно обработанным — защита от ошибочной маршрутизации в IdS |
| 6 | NOT_FOUND — entity_id не существует (логическая ошибка — PartyId исчез между Worker #2 и #6) | Нет | Пометить как failed → DLQ. Логировать NOT_FOUND |
| 6 | IDEMPOTENCY_DUPLICATE — ключ обработан конкурирующим инстансом (race condition) | Нет | Игнорировать. Логировать IDEMPOTENCY_DUPLICATE. Завершение без ошибки |
| 6 | CONCURRENT_MODIFICATION — version_uuid устарел (другой инстанс уже записал новую версию с более новым HLC) | Да — <!-- parameters: parties.party.retry_max_count --> 3 retry, <!-- parameters: parties.party.retry_backoff_ms --> 100ms начальная задержка, exponential backoff | Перечитать entity (повторить Worker #2), выполнить PATCH-merge с новой актуальной версией (Worker #4), сгенерировать новый HLC (Worker #5), повторить INSERT (Worker #6). После <!-- parameters: parties.party.retry_max_count --> 3 неудач — событие → DLQ |
| 6 | Сбой БД (INSERT failure, connectivity loss) | Да — <!-- parameters: parties.party.retry_max_count --> 3 retry, <!-- parameters: parties.party.retry_backoff_ms --> 100ms начальная задержка, exponential backoff | После <!-- parameters: parties.party.retry_max_count --> 3 неудач — событие → DLQ |
| 7 | Ошибка записи в outbox | Да — в рамках транзакции Workers 6–7 (полный откат при сбое) | Откат всей транзакции. Retry с начала (Worker #1–7). После 3 неудач — DLQ |
| Outbox relay | Брокер сообщений недоступен | Нет — outbox гарантирует доставку | Outbox-запись остается со статусом pending. Фоновый outbox relay retry-ит публикацию |

---

## Side Effects & Emitted Events

| Событие | После Worker (#) | Получатель |
|---------|-----------------|------------|
| PartyCacheUpdated | 7 (запись в outbox) / outbox relay (публикация) | TBD — будет определён на этапе проектирования C4 интеграций |

**Данные события:** partyId, dynamicAttributes (обновлённый набор — результат PATCH-merge), cacheStatus (текущее значение — контекст), updatedAt

---

## Implementation Targets

| Target | Назначение | Расположение |
|--------|-----------|-------------|
| UpdatePartyEventHandler | Принимает событие PartyProfileUpdatedEvent из IdS (через брокер сообщений) | parties/api/handlers/ |
| UpdatePartyUseCase | Бизнес-логика: валидация, загрузка Party, PATCH-мерж DynamicAttributes, запись в ledger + outbox | parties/app/use-cases/ |
| PartyLedgerRepository | Доступ к entity_ledger: чтение актуальной версии по HLC total order, INSERT (Append-Only) с проверкой unique constraint (entity_id, idempotency_key) и optimistic locking | parties/domain/ |
| HLCProvider | Генерация 4-компонентного HLC (hlc_l, hlc_c, hlc_instance_id, hlc_dc_id) для каждой ledger-записи | parties/infra/hlc/ |
| PartyEventPublisher | Публикация доменного события PartyCacheUpdated через outbox pattern | parties/infra/events/ |
