---
version: "1.0.3"
updated: "2026-05-24"
generated-from:
  - template: "templates-feat/180-tech-flow/tech-flow-template.md"
    feature: "tech-flow"
    agent: "architect"
    at: "2026-05-22"
    patch: false
  - template: "templates-feat/180-tech-flow/tech-flow-template.md"
    feature: "tech-flow (db-patterns alignment)"
    agent: "architect"
    at: "2026-05-24"
    patch: true
---

# Technical Flow: RegisterParty

**Entity:** party
**Связанные документы:** Use Case (uc-party-RegisterParty), Domain Model

---

## Входные данные

Операция запускается асинхронно по получению события `OrganizationCreated` от IdS (через брокер сообщений).

**DTO:** OrganizationCreatedEvent (из IdS)

| Поле | Тип | Обязательное | Описание |
|------|-----|-------------|----------|
| PartyId | UUID (string) | Да | Organization ID из IdS. Идентификатор организации |
| Inn | string | Да | ИНН организации: 10 цифр (юрлицо) или 12 цифр (ИП) |
| PartyName | string | Да | Название организации. Длина 1–200 символов |
| DynamicAttributes | jsonb | Нет | Динамический набор полей: ContactEmail, ContactPhone, BusinessSphere, Region, Ogrn, PersonalDataConsent, OfferAccepted и др. Состав определяется IdS |
| IdempotencyKey | UUID | Да | Ключ идемпотентности для защиты от дубликатов. Является `version_uuid` после записи (см. db-patterns) |

**Контекст вызова:**
- CorrelationId — сквозной идентификатор запроса (из сообщения брокера)
- IdempotencyKey — ключ идемпотентности для deduplication
- SourceSystem — IdS (внешняя система-источник)

---

## Порядок выполнения (Workers)

Цепочка Workers в порядке вызова. Операция выполняется синхронно внутри обработчика события до записи в outbox.

| # | Worker | Действие | Вход | Выход |
|---|--------|----------|------|-------|
| 1 | IdempotencyValidator | Проверить idempotency-ключ события; если уже обработан — завершить с пропуском | OrganizationCreatedEvent (IdempotencyKey) | void (или skip) |
| 2 | PartyExistenceValidator | Проверить, что Party с указанным PartyId не существует (защита от коллизии) | OrganizationCreatedEvent (PartyId) | void |
| 3 | MandatoryFieldsValidator | Проверить наличие всех mandatory полей: PartyId, Inn, PartyName. RegistrationType и CacheStatus не входят в payload события — устанавливаются системой в Worker #6 | OrganizationCreatedEvent | void |
| 4 | PartyDataValidator | Проверить формат ИНН (10 или 12 цифр), уникальность ИНН в системе, PartyName (не пустой, 1–200 символов) | OrganizationCreatedEvent (Inn, PartyName) | void |
| 5 | DynamicAttributesProcessor | Проверить наличие PersonalDataConsent и OfferAccepted в DynamicAttributes; если отсутствуют — записать предупреждение в лог (advisory, не блокирует) | OrganizationCreatedEvent (DynamicAttributes) | void |
| 6 | PartyAggregateCreator + HLCGenerator | Создать агрегат Party с CacheStatus=active, RegistrationType=real. Сгенерировать HLC (hlc_l, hlc_c, hlc_instance_id, hlc_dc_id) для ledger-записи. version_uuid = idempotency_key | OrganizationCreatedEvent (все поля) | Party aggregate (version_uuid, hlc_l, hlc_c, hlc_instance_id, hlc_dc_id) |
| 7 | PartyLedgerWriter | INSERT в entity_ledger (entity_id, version_uuid, hlc_l, hlc_c, hlc_instance_id, hlc_dc_id, origin_dc, idempotency_key, payload) + outbox-запись в рамках одной транзакции. Append-Only — только INSERT. Актуальная версия = max HLC по total order | Party aggregate (version_uuid, HLC, payload) | void |
| 8 | PartyCacheCreatedEventEmitter (outbox relay) | Прочитать outbox-запись и опубликовать событие PartyCacheCreated в брокер сообщений | Outbox record | void |

---

## Транзакционные границы

| Worker (#) | В транзакции? | Комментарий |
|-----------|-------------|-------------|
| 1–7 | Да | Единая БД-транзакция: валидация, создание агрегата, INSERT в entity_ledger (Append-Only) + outbox-запись (атомарно). Вместо UPDATE — новая версия в ledger |
| 8 | Нет | Внешняя публикация события — outbox relay (фоновый процесс). Outbox-запись гарантирует at-least-once доставку |

---

## Гарантии консистентности

| Гарантия | Обеспечивается |
|----------|---------------|
| Atomicity | Workers 1–7 в одной БД-транзакции: INSERT в entity_ledger (Append-Only) + outbox-запись. В случае сбоя — полный откат. UPDATE запрещён |
| Consistency | Все структурные инварианты проверены до сохранения: уникальность PartyId, уникальность ИНН, формат ИНН, длина PartyName. Инварианты проверяются до ledger-записи |
| Isolation | Append-Only Ledger: total order через HLC (hlc_l, hlc_c, hlc_instance_id, hlc_dc_id), optimistic locking через сравнение expected_version_uuid с version_uuid последней записи по HLC total order. Актуальная версия = max(HLC) |
| Durability | После commit транзакции: запись в entity_ledger (INSERT), outbox-запись сохранена. Событие будет доставлено outbox relay. Каждая запись имеет полную историю (Append-Only) |
| Exactly-once обработка | IdempotencyKey ≡ version_uuid (один ключ — одна версия). Unique constraint (entity_id, idempotency_key) предотвращает дубликаты. Outbox гарантирует ровно одну публикацию события |

---

## Обработка ошибок и retry

| Worker (#) | Ошибка | Retry? | Действие |
|-----------|--------|--------|----------|
| 1 | Idempotency-ключ уже обработан (дубликат события) | Нет | Игнорировать событие. Завершение без ошибки |
| 2 | PartyId уже существует (коллизия идентификаторов) | Нет | Пометить событие как failed → DLQ. Логировать PartyIdConflict |
| 3 | Отсутствует одно или несколько mandatory полей | Нет | Пометить событие как failed → DLQ. Логировать ValidationError с перечнем полей |
| 4 | ИНН в неверном формате (не 10 и не 12 цифр) | Нет | Пометить событие как failed → DLQ. Логировать ValidationError |
| 4 | Party с таким ИНН уже существует | Нет | Пометить событие как failed → DLQ. Логировать DuplicateInn |
| 4 | PartyName пустой или превышает 200 символов | Нет | Пометить событие как failed → DLQ. Логировать ValidationError |
| 7 | NOT_FOUND (entity_id не существует) | Нет | DLQ. Логировать NOT_FOUND. Указывает на логическую ошибку (PartyId отсутствует в ledger) |
| 7 | IDEMPOTENCY_DUPLICATE (ключ уже обработан) | Нет | Игнорировать. Логировать IDEMPOTENCY_DUPLICATE. Завершение без ошибки |
| 7 | CONCURRENT_MODIFICATION (version_uuid устарел — ожидаемый expected_version_uuid не совпал с текущей версией) | Да — <!-- parameters: parties.party.retry_max_count --> 3 retry, <!-- parameters: parties.party.retry_backoff_ms --> 100ms начальная задержка, exponential backoff | Перечитать entity, повторить попытку INSERT. После 3 неудач — событие → DLQ |
| 7 | Сбой БД (INSERT failure, connectivity loss) | Да — 3 retry, 100ms начальная задержка, exponential backoff | После 3 неудач — событие → DLQ |
| 8 | Брокер сообщений недоступен | Нет — outbox гарантирует доставку | Outbox-запись остаётся со статусом pending. Фоновый outbox relay retry-ит публикацию |

---

## Side Effects & Emitted Events

| Событие | После Worker (#) | Получатель |
|---------|-----------------|------------|
| PartyCacheCreated | 7 (запись в outbox) / 8 (публикация) | TBD — будет определён на этапе проектирования C4 интеграций |

**Данные события:** partyId, inn, partyName, cacheStatus=active, registrationType=real, dynamicAttributes, createdAt

---

## Cross-service Dependencies

Операция RegisterParty (создание кэша Party) является частью более широкого бизнес-процесса регистрации предприятия, который включает **привязку Admin User** (назначение администратора предприятия).

**Статус:** привязка Admin User **выходит за границы** parties/party и является ответственностью parties/user.

**Проблема (gap):**
- Domain Model (party §178, user §74) и BR-001 §47 указывают, что при `OrganizationCreated` должна выполняться **привязка Admin User**
- parties/party не получает UserId в payload события `OrganizationCreated` — физически не может выполнить привязку
- В parties/user **отсутствует** use case и tech-flow для обработки `OrganizationCreated` (есть только `DeactivateUserCacheFromIdS`)

**Требуется (отдельная задача):** создать use case `RegisterUserFromOrganization` и соответствующий tech-flow в parties/user для привязки Admin User при получении `OrganizationCreated`.

---

## Implementation Targets

| Target | Назначение | Расположение |
|--------|-----------|-------------|
| RegisterPartyEventHandler | Принимает событие OrganizationCreated из IdS (через брокер сообщений) | parties/api/handlers/ |
| RegisterPartyUseCase | Бизнес-логика: валидация, создание агрегата, сохранение, запись в outbox | parties/app/use-cases/ |
| PartyLedgerRepository | Доступ к entity_ledger: INSERT (Append-Only), чтение актуальной версии по HLC total order, проверка unique constraint (entity_id, idempotency_key) | parties/domain/ |
| HLCProvider | Генерация 4-компонентного HLC (hlc_l, hlc_c, hlc_instance_id, hlc_dc_id) для каждой ledger-записи | parties/infra/hlc/ |
| PartyEventPublisher | Публикация доменного события PartyCacheCreated через outbox pattern | parties/infra/events/ |
