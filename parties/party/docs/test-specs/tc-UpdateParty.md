---
version: "1.0.0"
updated: "2026-05-24"
agent: "architect"
status: draft
generated-from:
  - template: "templates-feat/185-test-spec/test-spec-template.md"
    feature: "test-spec"
    agent: "architect"
    at: "2026-05-24"
    patch: false
---

# Test Specification: UpdateParty

**Entity:** party
**Связанные документы:** [use-case](../use-cases/uc-party-UpdateParty.md), [domain-model](../domain-model.md), [security-policy](../../../docs/biz/security-policy.md)

> **Важно:** UpdateParty — async-обработчик события обновления профиля организации из IdS (через брокер сообщений). Не является пользовательским REST API. Все тесты проверяют обработку входящего события, а не HTTP-запросов. Формат ответа — логирование результата обработки: `processed` (успех), `skipped` (idempotency / empty payload), `failed` (ошибка → DLQ).
>
> **Операция:** частичное обновление DynamicAttributes с PATCH-семантикой. Mandatory-поля (Inn, PartyName, RegistrationType, PartyId, CacheStatus) не изменяются через этот UC — они задаются при создании (RegisterParty). CacheStatus изменяется через отдельные UC: DeactivatePartyCache / ReactivatePartyCache.

---

## Test Case Catalog

| ID | Тип | Источник (AC / правило) | Описание |
|----|-----|------------------------|----------|
| TC-001-U | happy | UC основной поток: успешное обновление DynamicAttributes | Обработать валидное событие обновления профиля → DynamicAttributes обновлены, PartyCacheUpdated опубликовано |
| TC-002-U | negative | UC alt-1: дубликат idempotency-ключа | Событие с уже обработанным idempotency-ключом → skip (игнорирование) |
| TC-003-U | negative | UC alt-2: Party не найдена | PartyId из события не существует → DLQ |
| TC-004-U | negative | UC alt-3: payload не содержит DynamicAttributes (пустой) | Payload без DynamicAttributes → skip (игнорирование) |
| TC-005-U | negative | UC alt-4: ошибка сохранения БД / конкурентный доступ | Сбой при записи → retry 3x → DLQ |
| TC-006-U | negative | UC alt-5: ошибка публикации outbox-события | Outbox-запись остаётся со статусом pending |
| TC-007-U | negative | UC alt-6: idempotency-ключ существует, но payload не совпадает | Replay-атака с изменённым payload → DLQ |
| TC-008-U | negative | UC alt-7: payload содержит Mandatory-поля (недопустимые для этого UC) | Запрещённые Mandatory-поля игнорируются, DynamicAttributes обновляются |
| TC-009-U | negative | domain-model: HLC total order — конфликт двух версий | Актуальное состояние — запись с максимальным HLC по total order |
| TC-010-U | negative | domain-model: Ledger idempotency | Повторная отправка с тем же idempotency_key не создаёт дубликат версии в ledger |
| TC-011-U | negative | domain-model: Optimistic locking — неверный version_uuid | CONCURRENT_MODIFICATION при попытке записи по неверному version_uuid |
| TC-012-U | negative | domain-model: Ledger append — иммутабельность | Новая версия добавлена, старая версия НЕ изменена |

---

## Happy Path Tests

### TC-001-U: Успешное обновление DynamicAttributes (PATCH-семантика)

| Given | When | Then |
|-------|------|------|
| Party с указанным PartyId существует. Idempotency-ключ события уникален (не обработан ранее). Payload содержит DynamicAttributes с частичным набором полей | Система обрабатывает событие обновления профиля из IdS | DynamicAttributes в Party обновлены: значения из payload перезаписали соответствующие ключи. Ключи, отсутствующие в payload, не изменены (PATCH-семантика). PartyCacheUpdated опубликовано. Party доступна в реестре с обновлёнными данными. Статус: processed |

**Steps:**
1. IdempotencyValidator — ключ уникален → passed
2. PartyLoader — Party загружена по PartyId → found
3. DynamicAttributesValidator — payload содержит DynamicAttributes (не пустой) → passed
4. MandatoryFieldsFilter — запрещённые Mandatory-поля (Inn, PartyName, RegistrationType, PartyId, CacheStatus) отфильтрованы → passed
5. DynamicAttributesMerger — выполнен merge с PATCH-семантикой: новые ключи записаны, существующие перезаписаны, отсутствующие в payload не изменены
6. HLCGenerator — сгенерирован HLC (hlc_l, hlc_c, hlc_instance_id, hlc_dc_id) для новой версии ledger
7. PartyLedgerWriter — INSERT в entity_ledger + outbox-запись в рамках одной транзакции
8. PartyCacheUpdatedEventEmitter — событие опубликовано

**Stub:** `stub_update_valid` — Party существует (active), событие с валидным PartyId (UUID), DynamicAttributes = {BusinessSphere: "Новая сфера", Region: "77"}, IdempotencyKey (UUID, уникальный)

---

## Negative Tests

### TC-002-U: Дубликат idempotency-ключа (replay-защита)

| Given | When | Then |
|-------|------|------|
| Событие обновления профиля с IdempotencyKey, который уже был обработан ранее (идентичный payload) | IdempotencyValidator проверяет ключ | IdempotencyValidator обнаруживает дубликат. Событие игнорируется. Статус: skipped. DynamicAttributes не изменяются. PartyCacheUpdated не публикуется |

**Stub:** `stub_update_idempotency_duplicate` — событие с IdempotencyKey, совпадающим с уже обработанным

---

### TC-003-U: Party не найдена (PartyId не существует)

| Given | When | Then |
|-------|------|------|
| Событие обновления профиля с PartyId, который не существует в системе (нет Party с таким ID) | PartyLoader загружает Party по PartyId | Событие помечается как failed → DLQ. Логируется ошибка PartyNotFound — «Party с указанным ID не найдена». Требуется ручное разрешение (IdS — синхронизация данных) |

**Stub:** `stub_update_party_not_found` — событие с PartyId (UUID), для которого нет Party в ledger

---

### TC-004-U: Payload пустой (не содержит DynamicAttributes)

| Given | When | Then |
|-------|------|------|
| Событие обновления профиля, в котором DynamicAttributes пуст (null, пустой объект `{}` или поле отсутствует). IdempotencyKey уникален | DynamicAttributesValidator проверяет payload | Система пропускает событие (ничего не обновляется). Статус: skipped. Логируется предупреждение EmptyUpdatePayload — «Событие обновления не содержит DynamicAttributes для изменения». Событие считается успешно обработанным (не failed — штатная ситуация) |

**Stub:** `stub_update_empty_payload` — событие с DynamicAttributes = null (или пустой объект `{}`)

---

### TC-005-U: Ошибка сохранения Party (сбой БД / конкурентный доступ)

| Given | When | Then |
|-------|------|------|
| Все валидации пройдены, DynamicAttributes обновлены, но при записи в ledger происходит ошибка (конкурентный доступ, сбой БД) | PartyLedgerWriter выполняет INSERT | Система выполняет retry до <!-- parameters: parties.party.retry_max_count --> 3 раз с экспоненциальной задержкой (начальная <!-- parameters: parties.party.retry_backoff_ms --> 100ms). После <!-- parameters: parties.party.retry_max_count --> 3 неудач — ConcurrencyConflict, событие → DLQ |

**Stub:** `stub_update_db_failure` — событие, при обработке которого PartyLedgerWriter симулирует ошибку БД

---

### TC-006-U: Ошибка публикации outbox-события

| Given | When | Then |
|-------|------|------|
| Party обновлена (ledger-запись создана) и outbox-запись создана, но брокер сообщений недоступен | PartyCacheUpdatedEventEmitter пытается опубликовать событие | Outbox-запись остаётся со статусом pending. Фоновый outbox relay retry-ит публикацию. Событие будет доставлено at-least-once |

**Stub:** `stub_update_broker_unavailable` — событие, после обработки которого брокер сообщений симулирует недоступность

---

### TC-007-U: Replay-атака (IdempotencyKey совпадает, payload отличается)

| Given | When | Then |
|-------|------|------|
| IdempotencyKey уже обработан ранее, но payload текущего события отличается от payload первого события с этим ключом | IdempotencyValidator проверяет ключ и payload | Система обнаруживает несовпадение payload. Событие помечается как failed → DLQ. Логируется IdempotencyKeyMismatch — «Idempotency-ключ уже обработан, но payload отличается. Возможна replay-атака». Требуется ручное разрешение |

**Stub:** `stub_update_replay_attack` — событие с IdempotencyKey, уже обработанным ранее, но с другим DynamicAttributes payload

---

### TC-008-U: Payload содержит Mandatory-поля (недопустимые для UpdateParty)

| Given | When | Then |
|-------|------|------|
| Событие обновления профиля содержит DynamicAttributes для обновления, но также включает Mandatory-поля (CacheStatus, Inn, PartyName, RegistrationType, PartyId), которые не должны изменяться через этот UC | DynamicAttributesMerger обрабатывает payload | Система игнорирует запрещённые Mandatory-поля. DynamicAttributes из payload обновляются (только ключи внутри DynamicAttributes-объекта). Запрещённые поля не применяются к Party. Логируется предупреждение UnexpectedFieldsInPayload — «Payload содержит Mandatory-поля, недопустимые для UpdateParty (CacheStatus, Inn, PartyName, RegistrationType, PartyId). Эти поля проигнорированы». Событие считается успешно обработанным |

**Stub:** `stub_update_unexpected_mandatory` — событие с валидными DynamicAttributes, но payload также содержит CacheStatus=inactive и Inn=новое_значение

---

### TC-009-U: HLC total order — выбор актуальной версии по тотальному порядку

| Given | When | Then |
|-------|------|------|
| В ledger существуют две версии Party с одним entity_id: v1 (HLC_l=1000, HLC_c=0, instance=A, dc=dc1) и v2 (HLC_l=1000, HLC_c=1, instance=B, dc=dc1). v2 имеет более высокий HLC total order (1000:1 > 1000:0). Событие обновления приходит для Party | Система загружает актуальное состояние Party (по HLC total order) для валидации перед записью | Загружена версия v2 (максимальный HLC по total order: l → c → instance_id → dc_id). Старая версия v1 не влияет на обработку. Новая версия v3 записывается с HLC > v2 |

**Stub:** `stub_update_hlc_total_order` — Party с двумя версиями в ledger (разные HLC). Событие обновления обрабатывается на основе актуальной версии

---

### TC-010-U: Idempotency — повторная отправка с тем же idempotency_key не создаёт дубликат версии в ledger

| Given | When | Then |
|-------|------|------|
| Событие обновления с IdempotencyKey, который уже использован и записан как version_uuid в ledger (успешная обработка) | Система получает событие повторно | Ledger не создаёт новую запись. Система возвращает статус skipped. При проверке ledger — существует ровно одна версия с данным idempotency_key/version_uuid для этого entity_id |

**Stub:** `stub_update_ledger_idempotency` — событие с IdempotencyKey, уже существующим в ledger как version_uuid для данного Party

---

### TC-011-U: Optimistic locking — неверный version_uuid → CONCURRENT_MODIFICATION

| Given | When | Then |
|-------|------|------|
| Попытка записи Party в ledger с version_uuid, не совпадающим с последней версией (устаревшая ссылка). Например: последняя версия Party имеет version_uuid=A, запись приходит с ожидаемым version_uuid=B | PartyLedgerWriter выполняет optimistic lock check | Система выбрасывает CONCURRENT_MODIFICATION. Событие помечается как failed → DLQ. Логируется ConcurrencyConflict. Требуется повторное чтение актуальной версии и повторная попытка |

**Stub:** `stub_update_optimistic_lock_conflict` — запись с заведомо неверным expected_version_uuid

---

### TC-012-U: Ledger append — иммутабельность предыдущих версий

| Given | When | Then |
|-------|------|------|
| Для Party существует версия v1 в ledger. Обработано событие обновления, записана новая версия v2 того же Party | Система проверяет состояние ledger после записи v2 | Версия v1 НЕ изменена (те же поля, тот же payload, тот же HLC). Версия v2 добавлена как новая запись. Обе версии доступны для чтения. DynamicAttributes из события отражены в payload версии v2 |

**Stub:** `stub_update_ledger_append_verify` — событие, после обработки которого выполняется верификация иммутабельности предыдущих версий ledger'а

| Правило (domain-rules / use-case) | Тест ID | Stub |
|-----------------------------------|---------|------|
| Idempotency-ключ уникален (дубликат → skip) | TC-002-U | `stub_update_idempotency_duplicate` |
| Party с PartyId существует (не найдена → DLQ) | TC-003-U | `stub_update_party_not_found` |
| Payload содержит DynamicAttributes (пусто → skip) | TC-004-U | `stub_update_empty_payload` |
| Ошибка сохранения → retry | TC-005-U | `stub_update_db_failure` |
| Outbox-публикация — pending при недоступности брокера | TC-006-U | `stub_update_broker_unavailable` |
| IdempotencyKey + payload mismatch → replay-атака → DLQ | TC-007-U | `stub_update_replay_attack` |
| Mandatory-поля в payload игнорируются | TC-008-U | `stub_update_unexpected_mandatory` |
| HLC total order — побеждает версия с максимальным HLC | TC-009-U | `stub_update_hlc_total_order` |
| Idempotency — повтор не создаёт дубликат версии в ledger | TC-010-U | `stub_update_ledger_idempotency` |
| Optimistic locking — неверный version_uuid → CONCURRENT_MODIFICATION | TC-011-U | `stub_update_optimistic_lock_conflict` |
| Ledger append — новая версия добавляется, старая не изменяется | TC-012-U | `stub_update_ledger_append_verify` |

---

## Security Tests (System-to-System)

Операция UpdateParty — async-обработчик события обновления профиля из IdS. Пользовательская аутентификация/авторизация отсутствует. Security-тесты покрывают угрозы на уровне интеграции систем.

### Authentication (Source Verification)

| ID | Тест | Угроза | Given | When | Then |
|----|------|--------|-------|------|------|
| SEC-001-U | Отсутствует IdempotencyKey | Replay / Integrity | Событие обновления профиля без поля IdempotencyKey | Система проверяет наличие ключа | Событие помечается как failed → DLQ. Логируется ValidationError — «Отсутствует idempotency-ключ» |
| SEC-002-U | Replay-атака (изменённый payload) | Replay | Злоумышленник перехватил событие и отправляет его повторно с изменённым DynamicAttributes, но тем же IdempotencyKey | Система проверяет idempotency-ключ и payload | Если payload отличается от первого события с этим ключом — IdempotencyKeyMismatch, событие → DLQ (покрыто TC-007-U) |
| SEC-003-U | Сообщение от неизвестного источника | Spoofing | Событие получено от отправителя, не являющегося IdS (неизвестный источник) | Система верифицирует источник сообщения | Событие отбрасывается. Логируется UnknownSourceError. **Механизм:** TBD — будет определён на этапе инфраструктурной интеграции (HMAC / TLS mutual auth / IP whitelist) |
| SEC-004-U | Некорректный payload (malformed JSON) | Deserialization Error | Событие обновления содержит повреждённый JSON (битые данные, непарсируемый payload) | Система десериализует событие | Ошибка десериализации, событие помечается как failed → DLQ. Логируется DeserializationError |

---

## Boundary Tests

| Поле (из domain-model) | Constraint | Тест ID | Значение | Then |
|------------------------|------------|---------|----------|------|
| DynamicAttributes | любой jsonb | — | Любой валидный JSON-объект (в т.ч. пустой `{}`) → успешно обновляется | success — DynamicAttributes обновлены |
| DynamicAttributes | null (поле отсутствует) | TC-004-U | DynamicAttributes = null или отсутствует | skip — предупреждение EmptyUpdatePayload |
| DynamicAttributes | значения-инъекции | — | DynamicAttributes = {"name": "<script>alert(1)</script>"} | success — parties не валидирует формат полей в DynamicAttributes (ответственность IdS). Значение кэшируется как есть |

> **Важно:** В отличие от RegisterParty, UpdateParty не принимает Mandatory-поля (Inn, PartyName) — они не подлежат boundary-тестированию в рамках этого UC. DynamicAttributes не валидируются parties по формату (ответственность IdS, domain-model §87).

---

## Stub ↔ Test Mapping

| Stub | Описание | Обслуживает тесты |
|------|----------|-------------------|
| `stub_update_valid` | Party существует (active). Событие с валидным PartyId, DynamicAttributes = {BusinessSphere: "Новая сфера"}, уникальный IdempotencyKey | TC-001-U |
| `stub_update_idempotency_duplicate` | Событие с IdempotencyKey, совпадающим с уже обработанным (идентичный payload) | TC-002-U |
| `stub_update_party_not_found` | Событие с PartyId (UUID), для которого нет Party в ledger | TC-003-U |
| `stub_update_empty_payload` | Событие с DynamicAttributes = null (или пустой объект `{}`) | TC-004-U |
| `stub_update_db_failure` | Событие, при обработке которого PartyLedgerWriter симулирует ошибку БД | TC-005-U |
| `stub_update_broker_unavailable` | Событие, после обработки которого брокер сообщений симулирует недоступность | TC-006-U |
| `stub_update_replay_attack` | Событие с IdempotencyKey, уже обработанным, но с другим DynamicAttributes payload | TC-007-U |
| `stub_update_unexpected_mandatory` | Событие с валидными DynamicAttributes + запрещённые Mandatory-поля (CacheStatus, Inn) | TC-008-U |
| `stub_update_hlc_total_order` | Party с двумя версиями в ledger (HLC_l=1000, HLC_c=0 и HLC_l=1000, HLC_c=1). Событие обрабатывается на основе актуальной версии | TC-009-U |
| `stub_update_ledger_idempotency` | Событие с IdempotencyKey, уже записанным как version_uuid в ledger для данного Party | TC-010-U |
| `stub_update_optimistic_lock_conflict` | Запись Party с expected_version_uuid, не совпадающим с последней версией в ledger | TC-011-U |
| `stub_update_ledger_append_verify` | Событие, после обработки которого проверяется, что предыдущая версия Party в ledger не изменилась | TC-012-U |
| — | Без стаба — тест проверяет инфраструктурный уровень (верификация источника, десериализация) | SEC-001-U, SEC-002-U, SEC-003-U, SEC-004-U |

---

## Чек-лист (выполнено)

- [x] Каждый альтернативный поток из UC покрыт минимум одним тестом (TC-002-U–TC-008-U)
- [x] Основной поток (Happy Path) покрыт (TC-001-U)
- [x] Каждая угроза из security-policy покрыта security-тестом (Spoofing, Replay, Deserialization)
- [x] Idempotency покрыта (TC-002-U — дубликат ключа, TC-007-U — replay-атака, TC-010-U — ledger idempotency)
- [x] Ledger-тесты: HLC total order (TC-009-U), idempotency (TC-010-U), optimistic locking (TC-011-U), append-immutability (TC-012-U) — все покрыты
- [x] Party not found → DLQ покрыт (TC-003-U)
- [x] Empty payload → skip покрыт (TC-004-U)
- [x] Stub ↔ Test Mapping не имеет дубликатов (один stub на несколько тестов — OK)
- [x] Параметры (`retry_max_count`, `retry_backoff_ms`) обёрнуты в `<!-- parameters: ... -->`
- [x] Mandatory-поля не тестируются как изменяемые — они запрещены для UpdateParty (TC-008-U проверяет игнорирование)
- [x] DynamicAttributes не тестируются на формат — валидация формата ответственность IdS (domain-model §87)

---

## Известные gaps (out of scope)

| Gap | Описание | Документ-источник | Покрывается |
|-----|----------|-------------------|-------------|
| Интеграционное тестирование с IdS | End-to-end тест: IdS отправляет событие → parties обрабатывает → данные обновлены в реестре | use-case, tech-flow | Отдельная задача: e2e-тестирование интеграции с IdS |
| Верификация источника сообщения | Механизм аутентификации источника событий (HMAC, TLS mutual auth, IP whitelist) не определён | security-policy | Будет определён на этапе инфраструктурной интеграции |
