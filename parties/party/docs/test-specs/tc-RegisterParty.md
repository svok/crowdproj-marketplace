---
version: "1.0.4"
updated: "2026-05-24"
generated-from:
  - template: "templates-feat/185-test-spec/test-spec-template.md"
    feature: "test-spec"
    agent: "architect"
    at: "2026-05-22"
    patch: false
  - template: "templates-feat/185-test-spec/test-spec-template.md"
    feature: "test-spec"
    agent: "architect"
    at: "2026-05-24"
    patch: true
    reason: "Fix: removed TC-022/TC-023 (RegistrationType, CacheStatus — system-set, not incoming fields); added TC-025 (HLC total order), TC-026 (Ledger idempotency), TC-027 (Optimistic locking), TC-028 (Ledger append)"
---

# Test Specification: RegisterParty

**Entity:** party
**Связанные документы:** [BR-001](../../../docs/biz/requirements/BR-001.md), [use-case](../use-cases/uc-party-RegisterParty.md), [domain-model](../domain-model.md), [tech-flow](../tech-flows/party-RegisterParty-impl.md), [security-policy](../../../docs/biz/security-policy.md)

> **Важно:** RegisterParty — async-обработчик события `OrganizationCreated` из IdS (через брокер сообщений). Не является пользовательским REST API. Все тесты проверяют обработку входящего события, а не HTTP-запросов. Формат ответа — логирование результата обработки: `processed` (успех), `skipped` (idempotency), `failed` (ошибка → DLQ).
>
> **Note:** RegisterParty — внутренний async-обработчик события IdS. Регистрация предприятия на UI-уровне (BR-001) выполняется в IdS. parties/party лишь кэширует результат.

---

## Test Case Catalog

| ID | Тип | Источник (AC / правило) | Описание |
|----|-----|------------------------|----------|
| TC-001 | happy | AC-1 (BR-001): успешная регистрация предприятия | Обработать валидное событие OrganizationCreated → Party создана |
| TC-002 | negative | UC alt-1: дубликат idempotency-ключа | Событие с уже обработанным idempotency-ключом → skip |
| TC-003 | negative | UC alt-2: коллизия PartyId | PartyId уже существует, ключ idempotency не совпадает → DLQ |
| TC-004 | negative | UC alt-3: отсутствуют mandatory поля | Поле PartyId, Inn или PartyName отсутствует → DLQ |
| TC-005 | negative | UC alt-4: неверный формат ИНН | ИНН не 10 и не 12 цифр → DLQ |
| TC-006 | negative | UC alt-5: дубликат ИНН | Party с таким ИНН уже существует → DLQ |
| TC-007 | negative | UC alt-6: PartyName пустой | PartyName отсутствует или пуст → DLQ |
| TC-008 | negative | UC alt-6: PartyName >200 символов | PartyName превышает 200 символов → DLQ |
| TC-009 | negative | UC alt-7: ошибка сохранения БД | Конкурентный доступ или сбой БД → retry 3x → DLQ |
| TC-010 | boundary | Inn: min=10 цифр | ИНН = 10 цифр → success |
| TC-011 | boundary | Inn: max=12 цифр | ИНН = 12 цифр → success |
| TC-012 | boundary | Inn: 9 цифр (min-1) | ИНН = 9 цифр → error |
| TC-013 | boundary | Inn: 13 цифр (max+1) | ИНН = 13 цифр → error |
| TC-014 | boundary | PartyName: min=1 символ | PartyName = 1 символ → success |
| TC-015 | boundary | PartyName: max=200 символов | PartyName = 200 символов → success |
| TC-016 | boundary | PartyName: 201 символ (max+1) | PartyName = 201 символ → error |
| TC-017 | security | Отсутствует IdempotencyKey | Событие без idempotency-ключа → error |
| TC-018 | security | Replay-атака | То же событие повторно отправлено с изменённым payload → skip |
| TC-019 | security | Сообщение от неизвестного отправителя | Событие, подписанное неизвестным источником → отбрасывается |
| TC-020 | security | Некорректный payload (malformed) | JSON-сообщение повреждено → ошибка десериализации |
| TC-021 | negative | UC alt-8: ошибка публикации outbox-события | Outbox-запись остаётся со статусом pending |
| TC-024 | boundary | domain-model: DynamicAttributes = null | DynamicAttributes полностью отсутствует или null → успех с лог-предупреждением |
| TC-025 | negative | domain-model: HLC total order — конфликт двух версий | Актуальное состояние — запись с максимальным HLC по total order |
| TC-026 | negative | domain-model: Ledger idempotency | Повторная отправка с тем же idempotency_key не создаёт дубликат версии в ledger |
| TC-027 | negative | domain-model: Optimistic locking — неверный version_uuid | CONCURRENT_MODIFICATION при попытке записи по неверному version_uuid |
| TC-028 | negative | domain-model: Ledger append — иммутабельность | Новая версия добавлена, старая версия НЕ изменена |

---

## Happy Path Tests

### TC-001: Успешная регистрация предприятия (создание кэша Party)

| Given | When | Then |
|-------|------|------|
| Событие `OrganizationCreated` получено от IdS | Система обрабатывает событие | Party создана с CacheStatus=active, RegistrationType=real. Событие `PartyCacheCreated` опубликовано. Статус: processed |

**Steps:**
1. IdempotencyValidator — ключ уникален → passed
2. PartyExistenceValidator — PartyId не существует → passed
3. MandatoryFieldsValidator — PartyId, Inn, PartyName присутствуют → passed
4. PartyDataValidator — ИНН = 10 цифр, PartyName валидный (1–200 символов), ИНН уникален → passed
5. DynamicAttributesProcessor — проверена (или отсутствует) → предупреждение в лог при отсутствии PersonalDataConsent/OfferAccepted (advisory)
6. PartyAggregateCreator — агрегат создан
7. PartyRepositorySave — сохранён в одной транзакции с outbox
8. PartyCacheCreatedEventEmitter — событие опубликовано

**Stub:** `stub_valid_event` — полностью валидное событие OrganizationCreated: PartyId (UUID), Inn (10 цифр), PartyName (1–200 символов), DynamicAttributes (опционально), IdempotencyKey (UUID)

---

## Negative Tests

### TC-002: Дубликат idempotency-ключа (replay-защита)

| Given | When | Then |
|-------|------|------|
| Событие OrganizationCreated с IdempotencyKey, который уже был обработан ранее | Система получает событие | IdempotencyValidator обнаруживает дубликат. Событие игнорируется. Статус: skipped. Party не создаётся повторно |

**Stub:** `stub_idempotency_duplicate` — событие с IdempotencyKey, совпадающим с уже обработанным

---

### TC-003: Коллизия PartyId (идентификатор уже существует, idempotency не совпадает)

| Given | When | Then |
|-------|------|------|
| Событие OrganizationCreated с PartyId, который уже существует в системе, IdempotencyKey отличается | Система проверяет существование PartyId | Событие помечается как failed → DLQ. Логируется ошибка PartyIdConflict. Требуется ручное разрешение в IdS |

**Stub:** `stub_partyid_conflict` — событие с существующим PartyId и новым IdempotencyKey

---

### TC-004: Отсутствуют обязательные mandatory-поля (PartyId, Inn, PartyName)

| Given | When | Then |
|-------|------|------|
| Событие OrganizationCreated, в котором отсутствует одно или несколько обязательных полей: PartyId, Inn, PartyName | MandatoryFieldsValidator проверяет payload | Событие помечается как failed → DLQ. Логируется ValidationError с перечнем отсутствующих полей |

**Stub:** `stub_missing_fields` — событие без поля PartyId (и/или Inn, PartyName)

---

### TC-005: Неверный формат ИНН

| Given | When | Then |
|-------|------|------|
| Событие OrganizationCreated с Inn в неверном формате (не 10 и не 12 цифр, например 9 цифр) | PartyDataValidator проверяет формат ИНН | Событие помечается как failed → DLQ. Логируется ValidationError — «Неверный формат ИНН (ожидается 10 или 12 цифр)» |

**Stub:** `stub_invalid_inn_format` — событие с Inn = «123456789» (9 цифр)

---

### TC-006: Дубликат ИНН (предприятие с таким ИНН уже существует)

| Given | When | Then |
|-------|------|------|
| Событие OrganizationCreated с Inn, который уже существует в системе | PartyDataValidator проверяет уникальность ИНН | Событие помечается как failed → DLQ. Логируется DuplicateInn — «Предприятие с таким ИНН уже зарегистрировано». Требуется ручное разрешение (IdS) |

**Stub:** `stub_duplicate_inn` — событие с Inn, совпадающим с существующей записью Party

---

### TC-007: PartyName пустой

| Given | When | Then |
|-------|------|------|
| Событие OrganizationCreated с пустым PartyName (null или пустая строка) | PartyDataValidator проверяет PartyName | Событие помечается как failed → DLQ. Логируется ValidationError — «Название предприятия обязательно, длина 1–200 символов» |

**Stub:** `stub_empty_name` — событие с PartyName = «» (пустая строка)

---

### TC-008: PartyName превышает 200 символов

| Given | When | Then |
|-------|------|------|
| Событие OrganizationCreated с PartyName длиной 201 символ | PartyDataValidator проверяет PartyName | Событие помечается как failed → DLQ. Логируется ValidationError — «Название предприятия обязательно, длина 1–200 символов» |

**Stub:** `stub_oversized_name` — событие с PartyName = строка из 201 символа

---

### TC-009: Ошибка сохранения Party (сбой БД / конкурентный доступ)

| Given | When | Then |
|-------|------|------|
| Все валидации пройдены, но при сохранении Party происходит ошибка (конкурентный доступ, сбой БД) | PartyRepositorySave выполняет запись | Система выполняет retry до <!-- parameters: parties.party.retry_max_count --> 3 раз с экспоненциальной задержкой (начальная <!-- parameters: parties.party.retry_backoff_ms --> 50ms). После 3 неудач — ConcurrencyConflict, событие → DLQ |

**Stub:** `stub_db_failure` — событие, при обработке которого PartyRepositorySave симулирует ошибку БД

---

### TC-024: DynamicAttributes = null (advisory, не блокирует)

| Given | When | Then |
|-------|------|------|
| Событие OrganizationCreated с DynamicAttributes = null (или поле отсутствует) | DynamicAttributesProcessor проверяет наличие PersonalDataConsent и OfferAccepted | Предупреждение в лог: «PersonalDataConsent и OfferAccepted отсутствуют — считается несогласованными». Party создаётся без ошибки |

**Stub:** `stub_null_dynamic_attrs` — событие с DynamicAttributes = null

---

### TC-021: Ошибка публикации outbox-события

| Given | When | Then |
|-------|------|------|
| Party сохранена и outbox-запись создана, но брокер сообщений недоступен | PartyCacheCreatedEventEmitter пытается опубликовать событие | Outbox-запись остаётся со статусом pending. Фоновый outbox relay retry-ит публикацию. Событие будет доставлено at-least-once |

**Stub:** `stub_broker_unavailable` — событие, после обработки которого брокер сообщений симулирует недоступность

---

### TC-025: HLC total order — выбор актуальной версии по тотальному порядку

| Given | When | Then |
|-------|------|------|
| В ledger существуют две версии Party с одним entity_id: v1 (HLC_l=1000, HLC_c=0, instance=A, dc=dc1) и v2 (HLC_l=1000, HLC_c=1, instance=B, dc=dc1). v2 имеет более высокий HLC total order (1000:1 > 1000:0) | Система читает актуальное состояние Party по entity_id | Актуальным считается состояние из версии v2 (максимальный HLC по total order: l → c → instance_id → dc_id). Старая версия v1 доступна как история |

**Stub:** `stub_hlc_total_order` — две версии Party в ledger с разными HLC

---

### TC-026: Idempotency — повторная отправка с тем же idempotency_key не создаёт дубликат версии в ledger

| Given | When | Then |
|-------|------|------|
| Событие OrganizationCreated с IdempotencyKey, который уже использован и записан как version_uuid в ledger (успешная обработка) | Система получает событие повторно | Ledger не создаёт новую запись. Система возвращает статус skipped. При проверке ledger — существует ровно одна версия с данным idempotency_key/version_uuid для этого entity_id |

**Stub:** `stub_ledger_idempotency` — событие с IdempotencyKey, уже существующим в ledger

---

### TC-027: Optimistic locking — неверный version_uuid → CONCURRENT_MODIFICATION

| Given | When | Then |
|-------|------|------|
| Попытка записи Party в ledger с version_uuid, не совпадающим с последней версией (устаревшая ссылка). Например: последняя версия Party имеет version_uuid=A, запрос приходит с version_uuid=B | PartyRepositorySave выполняет optimistic lock check | Система выбрасывает CONCURRENT_MODIFICATION. Событие помечается как failed → DLQ. Логируется ConcurrencyConflict. Требуется повторное чтение актуальной версии и повторная попытка |

**Stub:** `stub_optimistic_lock_conflict` — запись с заведомо неверным version_uuid

---

### TC-028: Ledger append — иммутабельность предыдущих версий

| Given | When | Then |
|-------|------|------|
| Для Party существует версия v1 в ledger. Записана новая версия v2 того же Party | Система проверяет состояние ledger после записи v2 | Версия v1 НЕ изменена (те же поля, тот же payload, тот же HLC). Версия v2 добавлена как новая запись. Обе версии доступны для чтения |

**Stub:** `stub_ledger_append_verify` — событие, после обработки которого выполняется верификация иммутабельности предыдущих версий ledger'а

| Правило (domain-rules / use-case) | Тест ID | Stub |
|-----------------------------------|---------|------|
| Idempotency-ключ уникален | TC-002 | `stub_idempotency_duplicate` |
| PartyId не существует (коллизия) | TC-003 | `stub_partyid_conflict` |
| Все mandatory поля присутствуют | TC-004 | `stub_missing_fields` |
| Формат ИНН: 10 или 12 цифр | TC-005 | `stub_invalid_inn_format` |
| ИНН уникален в системе | TC-006 | `stub_duplicate_inn` |
| PartyName не пустой | TC-007 | `stub_empty_name` |
| PartyName 1–200 символов | TC-008 | `stub_oversized_name` |
| Ошибка сохранения → retry | TC-009 | `stub_db_failure` |
| Ошибка публикации outbox | TC-021 | `stub_broker_unavailable` |
| DynamicAttributes = null (advisory) | TC-024 | `stub_null_dynamic_attrs` |
| HLC total order — побеждает версия с максимальным HLC | TC-025 | `stub_hlc_total_order` |
| Idempotency — повтор не создаёт дубликат версии в ledger | TC-026 | `stub_ledger_idempotency` |
| Optimistic locking — неверный version_uuid → CONCURRENT_MODIFICATION | TC-027 | `stub_optimistic_lock_conflict` |
| Ledger append — новая версия добавляется, старая не изменяется | TC-028 | `stub_ledger_append_verify` |

---

## Security Tests (System-to-System)

Операция RegisterParty — async-обработчик события `OrganizationCreated` из IdS. Пользовательская аутентификация/авторизация отсутствует. Security-тесты покрывают угрозы на уровне интеграции систем.

### Authentication (Source Verification)

| ID | Тест | Угроза | Given | When | Then |
|----|------|--------|-------|------|------|
| SEC-001 | Отсутствует IdempotencyKey | Replay / Integrity | Событие OrganizationCreated без поля IdempotencyKey | Система проверяет ключ | Событие помечается как failed → DLQ. Логируется ValidationError — «Отсутствует idempotency-ключ» |
| SEC-002 | Replay-атака (другое событие с тем же ключом) | Replay | Злоумышленник перехватил событие и отправляет его повторно с изменённым payload, но тем же IdempotencyKey | Система проверяет idempotency-ключ | Если payload отличается от первого события с этим ключом — конфликт, событие → DLQ. Логируется IdempotencyKeyMismatch. **Расширение:** use-case alt-1 описывает только duplicate-skip (одинаковый payload); SEC-002 покрывает security-сценарий (разный payload) |
| SEC-003 | Сообщение от неизвестного источника | Spoofing | Событие получено от отправителя, не являющегося IdS (неизвестный источник) | Система верифицирует источник сообщения | Событие отбрасывается. Логируется UnknownSourceError. **Механизм:** TBD — будет определён на этапе инфраструктурной интеграции (HMAC / TLS mutual auth / IP whitelist) |
| SEC-004 | Некорректный payload (malformed JSON) | Deserialization Error | Событие OrganizationCreated содержит повреждённый JSON (битые данные, непарсируемый payload) | Система десериализует событие | Ошибка десериализации, событие помечается как failed → DLQ. Логируется DeserializationError |

---

## Boundary Tests

| Поле (из domain-model) | Constraint | Тест ID | Значение | Then |
|------------------------|------------|---------|----------|------|
| Inn | 10 цифр (юрлицо) | TC-010 | «1234567890» (10 цифр) | success — Party создана |
| Inn | 12 цифр (ИП) | TC-011 | «123456789012» (12 цифр) | success — Party создана |
| Inn | min-1: 9 цифр | TC-012 | «123456789» (9 цифр) | error — ValidationError: неверный формат ИНН |
| Inn | max+1: 13 цифр | TC-013 | «1234567890123» (13 цифр) | error — ValidationError: неверный формат ИНН |
| Inn | null | — | null | error — missing mandatory field (покрыто TC-004) |
| PartyName | min: 1 символ | TC-014 | «A» (1 символ) | success — Party создана |
| PartyName | max: 200 символов | TC-015 | Строка из 200 символов | success — Party создана |
| PartyName | max+1: 201 символ | TC-016 | Строка из 201 символа | error — ValidationError (покрыто TC-008) |
| PartyName | null/empty | — | null или «» | error — ValidationError (покрыто TC-007) |
| DynamicAttributes | null (полностью отсутствует) | TC-024 | null | success — Party создана, лог-предупреждение (advisory) |

---

## Stub ↔ Test Mapping

| Stub | Описание | Обслуживает тесты |
|------|----------|-------------------|
| `stub_valid_event` | Полностью валидное событие OrganizationCreated: PartyId (UUID), Inn (10 цифр), PartyName (1–200 символов), DynamicAttributes, IdempotencyKey (UUID) | TC-001, TC-010, TC-011, TC-014, TC-015 |
| `stub_idempotency_duplicate` | Событие с IdempotencyKey, совпадающим с уже обработанным | TC-002 |
| `stub_partyid_conflict` | Событие с PartyId, который уже существует, и новым IdempotencyKey | TC-003 |
| `stub_missing_fields` | Событие без одного или нескольких mandatory полей (PartyId, Inn, PartyName) | TC-004 |
| `stub_invalid_inn_format` | Событие с Inn = «123456789» (9 цифр) | TC-005, TC-012 |
| `stub_duplicate_inn` | Событие с Inn, совпадающим с существующей записью Party | TC-006 |
| `stub_empty_name` | Событие с PartyName = «» (пустая строка) | TC-007 |
| `stub_oversized_name` | Событие с PartyName = строка из 201 символа | TC-008, TC-016 |
| `stub_db_failure` | Событие, при обработке которого PartyRepositorySave симулирует ошибку БД | TC-009 |
| `stub_broker_unavailable` | Событие, после обработки которого брокер сообщений симулирует недоступность | TC-021 |
| `stub_null_dynamic_attrs` | Событие с DynamicAttributes = null | TC-024 |
| `stub_hlc_total_order` | Две версии Party в ledger: v1 (HLC_l=1000, HLC_c=0) и v2 (HLC_l=1000, HLC_c=1). Второй — более поздняя | TC-025 |
| `stub_ledger_idempotency` | Событие с IdempotencyKey, уже записанным как version_uuid в ledger | TC-026 |
| `stub_optimistic_lock_conflict` | Запись Party с version_uuid, не совпадающим с последней версией в ledger | TC-027 |
| `stub_ledger_append_verify` | Событие, после обработки которого проверяется, что предыдущая версия Party в ledger не изменилась | TC-028 |
| — | Без стаба — тест проверяет инфраструктурный уровень (верификация источника, десериализация) | SEC-001, SEC-002, SEC-003, SEC-004 |

---

## Чек-лист (выполнено)

- [x] Каждый AC из BR-001 покрыт минимум одним тестом (успешная регистрация, формат ИНН, обязательные поля — адаптировано для async)
- [x] Каждая угроза из security-policy покрыта security-тестом (Spoofing, Replay, Injection — адаптировано для system-to-system)
- [x] AuthN bypass покрыт (SEC-001 — отсутствие IdempotencyKey, SEC-003 — неизвестный источник)
- [x] Каждое правило валидации из domain-rules (structural invariants) покрыто негативным тестом (TC-002–TC-009, TC-021, TC-024–TC-028)
- [x] Для каждого constrained поля из entity-spec: boundary тест (min, max, min-1, max+1)
- [x] Stub ↔ Test Mapping не имеет дубликатов (один stub на несколько тестов — OK)
- [x] Параметры (`retry_max_count`, `retry_backoff_ms`) обёрнуты в `<!-- parameters: ... -->`
- [x] Ledger-тесты: HLC total order (TC-025), idempotency (TC-026), optimistic locking (TC-027), append-immutability (TC-028) — все покрыты
- [x] RegistrationType и CacheStatus не тестируются как входящие поля — они системные (Worker #6), TC-022 и TC-023 удалены

---

## Известные gaps (out of scope)

| Gap | Описание | Документ-источник | Покрывается |
|-----|----------|-------------------|-------------|
| Привязка Admin User | При `OrganizationCreated` должна выполняться привязка Admin User, но события IdS не содержат userId | tech-flow §112-116, BR-001 §47 | Отдельная задача: `parties/user/RegisterUserFromOrganization` — test-spec будет создан отдельно |
