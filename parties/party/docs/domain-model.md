---
version: "1.1.3"
updated: "2026-05-24"
agent: "architect"
status: approved
generated-from:
  - template: "templates-feat/140-domain-model/domain-model-template.md"
    feature: "domain-model"
    agent: "architect"
    at: "2026-05-21"
    patch: false
  - template: "templates-feat/140-domain-model/domain-model-template.md"
    feature: "domain-model"
    agent: "architect"
    at: "2026-05-21"
    patch: true
    reason: "Fundamental rework: Party is now a cache from IdS (not Source-of-Truth). Removed business State Machine (Active/Disabled/Blocked), replaced with CacheStatus (active/inactive). Added MVP+ evolution: fictitious records from crawler + matching by INN with proof of authority. Added PartyFictitiousCreated and PartyClaimed events."
  - template: "templates-feat/140-domain-model/domain-model-template.md"
    feature: "domain-model"
    agent: "designer"
    at: "2026-05-22"
    patch: true
    reason: "UX review: split data into Mandatory fields (structured) + DynamicAttributes (jsonb). IdS field set is dynamic — parties only validates mandatory fields (PartyId, Inn, PartyName, RegistrationType, CacheStatus). Removed hardcoded validations for ContactEmail/Phone/Region/Ogrn/consents — validation is IdS responsibility."
  - template: "templates-feat/140-domain-model/domain-model-template.md"
    feature: "domain-model"
    agent: "architect"
    at: "2026-05-24"
    patch: true
    reason: "Added Ledger Structure (Append-Only) section: entity_ledger schema, HLC 4-component (l, c, instance_id, dc_id), version_uuid as identity, idempotency_key, retention policy."
---

# Domain Model: party

**Связанные документы:** Decomposition, Global Domain Rules, BR-001, BR-002, BR-005, user/domain-model.md

---

## Aggregate: party (dual nature — кэш IdS / нативный для фиктивных)

Party в сервисе parties имеет **двойственную природу** в зависимости от источника данных:

| Источник данных | Source-of-Truth | Поведение |
|----------------|-----------------|-----------|
| **IdS** (MVP) | IdS | Party — локальный кэш данных организации из IdS. Данные предприятия (ИНН, регион, сфера деятельности, контакты) создаются через IdS, parties кэширует их для поиска/фильтрации |
| **Краулер** (MVP+) | parties | Party — нативная запись, созданная краулером. Не имеет учётной записи в IdS. Source-of-Truth для этой записи — parties |
| **Сопоставление** (MVP+) | мигрирует из parties → IdS | При регистрации организации в IdS с ИНН, совпадающим с фиктивной записью, Source-of-Truth переходит от parties к IdS. Party становится кэшем |

Parties хранит данные предприятия, необходимые для поиска и фильтрации в реестре предприятий и отображения в публичных карточках (BR-005). Управление жизненным циклом организации, зарегистрированной в IdS (создание, отключение, блокировка, обновление профиля) — полностью во внешнем IdS. Для фиктивных записей (MVP+) жизненный цикл управляется краулером внутри parties.

**Идентификатор:** PartyId (UUID). Для записей из IdS — Organization ID из IdS. Для фиктивных записей (MVP+) — UUID, сгенерированный parties при создании краулером
**Тип:** Aggregate Root
**Хранение:** state-sourced

---

## Структура данных: mandatory + dynamicAttributes

Party хранит данные предприятия в двух категориях:

### Mandatory fields (жёсткая схема)

Поля, которые всегда присутствуют в схеме данных. Хранятся как отдельные колонки.

| Поле | Тип | Правила валидации |
|------|-----|-------------------|
| PartyId | UUID (string) | Для записей из IdS — Organization ID из IdS. Для fictitious (MVP+) — UUID, сгенерированный parties при создании краулером |
| RegistrationType | enum | `real` — организация зарегистрирована через IdS (кэш). `auto` — создана автоматически краулером (фиктивная, MVP+). В MVP — всегда `real` |
| Inn | string | 10 цифр (ИНН юрлица) или 12 цифр (ИНН ИП). Не пустой. Уникальный в системе |
| PartyName | string | Название организации. Не пустой, длина 1–200 символов |
| CacheStatus | enum | Статус кэша: `active` | `inactive`. Отражает актуальность данных из IdS. Причины перехода между статусами — вне контекста parties (определяются IdS) |

### DynamicAttributes (semi-structured)

Остальные поля, состав которых **динамический** и настраивается в IdS. Хранятся как `jsonb` (или аналогичный semi-structured тип):

| Поле | Тип в jsonb | Комментарий |
|------|-------------|-------------|
| ContactEmail | string | RFC 5322. Передаётся из IdS, кэшируется в parties. Валидация формата — ответственность IdS |
| ContactPhone | string | E.164. Опциональный. Валидация — IdS |
| BusinessSphere | string | Сфера деятельности. Free-text. Опциональный |
| Region | string | Справочник субъектов РФ. Опциональный |
| Ogrn | string | 13 или 15 цифр. Валидация — IdS. Не публикуется в публичных API |
| PersonalDataConsent | boolean | Согласие на обработку ПД (152-ФЗ). Валидация — IdS |
| OfferAccepted | boolean | Принятие оферты. Валидация — IdS |
| /* и любые другие поля, переданные из IdS */ | any | parties не валидирует unknown-поля. Состав определяется настройкой профиля организации в IdS |

> **Важно:** Parties **не валидирует** формат полей в DynamicAttributes. Валидация формата — ответственность IdS (Source-of-Truth). Parties проверяет только обязательность и формат **mandatory-полей**. DynamicAttributes принимаются как есть и кэшируются для поиска/отображения.

> Parties не решает, какие поля показывать пользователю — это ответственность UI-слоя и IdS. Если в IdS настроено новое поле, parties кэширует его без изменения схемы.

---

## Cache Status (состояние кэша) — MVP

В MVP Party в parties — это кэш данных из IdS, не управляемая бизнес-сущность parties. Вместо State Machine (не применима — нет бизнес-логики переходов в parties) используется простой статус кэша:

| Статус | Описание |
|--------|----------|
| active | Данные актуальны, предприятие отображается в реестре и публичных карточках (BR-005). Объявления предприятия активны и видны в поиске |
| inactive | Организация отключена/заблокирована/удалена в IdS. **По умолчанию не выводится** в публичных UI-компонентах. **Доступна в исторических данных и поиске по истории.** Объявления предприятия скрыты. Организация никогда не удаляется физически (мягкое удаление через статус) |

> **Решение (MVP):** parties не реализует State Machine для party. Логика переходов между статусами организации — ответственность IdS. Parties получает конечный статус из IdS и хранит только `active`/`inactive`.

**Контракт маппинга статусов IdS → CacheStatus (parties/party):**

| Статус в IdS | CacheStatus в parties | Условие |
|--------------|----------------------|---------|
| active | active | Организация активна в IdS, кэш актуален для отображения в реестре |
| disabled | inactive | Организация отключена администратором — не показывать в публичных UI |
| blocked | inactive | Организация заблокирована модератором — не показывать в публичных UI |
| deleted | inactive | Организация удалена — не показывать в публичных UI |

---

## State Machine — MVP+ (эволюция)

Начиная с MVP+ в parties/party добавляется новый источник данных — **краулер**, который собирает данные об организациях из открытых источников (интернет). Такие записи являются **фиктивными** — они не имеют соответствующей учётной записи в IdS.

### Состояния (полная модель, включая MVP+)

| Состояние | Описание | Появляется |
|-----------|----------|------------|
| fictitious | Запись создана краулером на основе данных из открытых источников. Не имеет учётной записи в IdS. Отображается в реестре предприятий с пометкой «неподтверждённая организация» | MVP+ |
| active | Организация зарегистрирована в IdS, кэш активен. Либо фиктивная запись, связанная с реальной регистрацией в IdS. Отображается в реестре стандартно | MVP (кэш) / MVP+ (связывание) |
| inactive | Организация отключена/заблокирована/удалена в IdS. Не отображается в публичных UI | MVP |

### Разрешённые переходы

| Из | В | Триггер | Guard (условие) | Появляется |
|----|---|---------|-----------------|------------|
| fictitious | active | Регистрация организации в IdS + сопоставление по ИНН + подтверждение полномочий пользователем | ИНН фиктивной записи совпадает с ИНН из OrganizationCreated. Пользователь доказал полномочия управления организацией | MVP+ |
| active | inactive | Статусное событие из IdS (disabled/blocked/deleted) | — | MVP |
| inactive | active | Статусное событие из IdS (enabled/unblocked) | — | MVP |

### Логика сопоставления (MVP+)

При получении события `OrganizationCreated` из IdS, parties:
1. Проверяет наличие существующей записи Party с таким же ИНН (`Inn`).
2. **Если** запись найдена и её статус = `fictitious`:
   - Связывает Party с organization_id из IdS.
   - Переводит статус в `active`.
   - Обновляет кэшированные данные из payload события IdS. **Приоритет — данные из IdS** как верифицированные. Данные фиктивной записи — fallback для полей, отсутствующих в payload.
3. **Если** запись не найдена или её статус ≠ `fictitious`:
   - Создаёт новую запись Party со статусом `active` и связывает с organization_id из IdS.
4. **Подтверждение полномочий:** Пользователь, регистрирующий организацию, должен подтвердить, что он уполномочен ею управлять (механизм верификации — вне контекста parties, определяется в BR-001 / use-case регистрации).

> **Важно:** В MVP фиктивные записи отсутствуют — все Party создаются только по событиям из IdS со статусом `active`. Функция краулера и сопоставления — MVP+ и пост-MVP.

---

## Structural Invariants

Правила, истинные ВСЕГДА для агрегата, в любом состоянии.

| Инвариант | Комментарий |
|-----------|-------------|
| PartyId != null | Для записей из IdS — Organization ID из IdS. Для fictitious (MVP+) — UUID, сгенерированный parties |
| Inn — 10 или 12 цифр | ИНН юрлица (10) или ИП (12). Форматный контроль при создании кэша |
| Inn уникален в системе | Два предприятия не могут иметь одинаковый ИНН. Уникальность проверяется по всем записям, включая fictitious |
| PartyName не пустой (1–200 символов) | Название обязательно для отображения в реестре |
| CacheStatus ∈ {active, inactive} (MVP) | В MVP кэш может быть только активным или неактивным. Статус `fictitious` появляется в MVP+ |
| RegistrationType ∈ {real, auto} | `real` — регистрация через IdS. `auto` — автоматическое создание краулером (MVP+) |
| DynamicAttributes — без валидации формата в parties | Валидация формата полей внутри DynamicAttributes — ответственность IdS (Source-of-Truth). Parties кэширует данные как есть |
| Контактные данные не отображаются публично | ContactEmail и ContactPhone не показываются в реестре предприятий и публичных карточках согласно BR-005. Решение о видимости — UI-слой, не parties |
| Party никогда не удаляется физически | Только мягкое удаление через статус `inactive` или в MVP+ через `fictitious` → удаление не предусмотрено. Hard-delete запрещён |

---

## Domain Events

События, которые публикует агрегат parties/party.

| Событие | Когда публикуется | Данные | Появляется |
|---------|------------------|--------|------------|
| PartyCacheCreated | При создании кэша Party (после получения OrganizationCreated из IdS) | partyId, inn, partyName, cacheStatus, registrationType=real, dynamicAttributes, createdAt | MVP |
| PartyCacheUpdated | При обновлении кэшированных данных предприятия из IdS | partyId, cacheStatus, dynamicAttributes (обновлённый набор), updatedAt | MVP |
| PartyCacheStatusChanged | При изменении статуса кэша (active ↔ inactive) по событию из IdS | partyId, previousStatus, newStatus, cacheStatus, changedAt | MVP |
| PartyFictitiousCreated | При создании фиктивной записи краулером | partyId, inn, partyName, registrationType=auto, dynamicAttributes, createdAt | MVP+ |
| PartyClaimed | При сопоставлении фиктивной записи с реальной регистрацией в IdS (fictitious → active) | partyId, inn, organizationId (из IdS), registrationType: auto→real, dynamicAttributes (из IdS), claimedBy, claimedAt | MVP+ |

**Триггеры (входящие события):**

От IdS (MVP):
- `OrganizationCreated` — создание организации в IdS → создание кэша Party + привязка Admin User. Набор полей (mandatory + dynamicAttributes) определяется payload'ом события из IdS
- Обновление профиля организации в IdS — обновление кэшированных данных Party. Состав полей динамический, parties обновляет то, что пришло в событии
- Статусное событие организации из IdS (enabled/disabled/blocked/deleted) — обновление cacheStatus в кэше Party

От краулера (MVP+):
- Создание записи краулером (внутренняя команда parties) → создание фиктивной Party со статусом `fictitious`

---

## Ledger Structure (Append-Only)

Все изменения агрегата Party фиксируются в Append-Only Ledger. Каждая версия Party — иммутабельная запись в ledger. Актуальное состояние — запись с максимальным HLC по total order.

### Identity

| Компонент identity | Тип | Назначение |
|-------------------|-----|------------|
| `entity_id` | UUID (string) | PartyId. Для записей из IdS — Organization ID из IdS. Для fictitious (MVP+) — UUID, сгенерированный parties |
| `version_uuid` | UUID | Уникальный идентификатор версии записи в ledger. Является частью составного identity `(entity_id, version_uuid)`. Каждое изменение Party создаёт новую версию с новым version_uuid. Совпадает с idempotency_key запроса, создавшего версию |

### Entity Ledger Schema

Поля, составляющие запись в ledger:

| Поле ledger | Тип | Назначение |
|-------------|-----|------------|
| `entity_id` | UUID | PartyId |
| `version_uuid` | UUID | Идентификатор версии. Входит в состав identity записи. Генерируется как UUID v4 на клиенте (или сервисе-инициаторе). Гарантирует уникальность версии |
| `hlc_l` | int64 | HLC logical time — миллисекунды от epoch, скорректированные по Hybrid Logical Clock |
| `hlc_c` | int32 | HLC counter — разрешение коллизий timestamps в пределах одной миллисекунды |
| `hlc_instance_id` | string | Уникальный ID инстанса, создавшего запись (runtime-метаданные: AWS_LAMBDA_LOG_STREAM_NAME / K_REVISION / HOSTNAME) |
| `hlc_dc_id` | string | ID дата-центра, в котором создана запись (runtime-метаданные: AWS_REGION / KUBERNETES_ZONE / env DC_ID) |
| `origin_dc` | string | Дата-центр, в котором запись впервые принята |
| `idempotency_key` | UUID | Ключ идемпотентности мутирующего запроса. После успешной записи становится `version_uuid` |
| `payload` | jsonb | Данные Party: mandatory поля (PartyId, Inn, PartyName, RegistrationType, CacheStatus) + dynamicAttributes |
| `is_tombstone` | boolean | Флаг мягкого удаления. `true` — запись помечена как удалённая (Party никогда не удаляется физически) |
| `version_vector` | map‹dc_id → sequence› | **Nullable.** В MVP — всегда NULL. Заполняется на этапе Multi-DC для обнаружения concurrent-изменений |

### Hybrid Logical Clock (HLC)

Party использует 4-компонентный HLC для тотального упорядочивания записей в ledger:

- **`hlc_l`** — logical time (ms от epoch, скорректированные по правилам HLC)
- **`hlc_c`** — counter (сбрасывается в 0 при каждом изменении `l`, инкрементируется при коллизиях в пределах ms)
- **`hlc_instance_id`** — идентификатор инстанса-создателя (генерируется при cold start, переиспользуется между warm invocations)
- **`hlc_dc_id`** — идентификатор дата-центра (константа для single-DC в MVP)

**Total order (строгий порядок):**

```
A ≺ B ⟺ A.hlc_l < B.hlc_l ∨
         (A.hlc_l = B.hlc_l ∧ A.hlc_c < B.hlc_c) ∨
         (A.hlc_l = B.hlc_l ∧ A.hlc_c = B.hlc_c ∧ A.hlc_instance_id < B.hlc_instance_id) ∨
         (A.hlc_l = B.hlc_l ∧ A.hlc_c = B.hlc_c ∧ A.hlc_instance_id = B.hlc_instance_id ∧ A.hlc_dc_id < B.hlc_dc_id)
```

**Генерация HLC (при создании нового события):**
```
l_new = max(current_state.l, wall_clock_ms)
c_new = (l_new == current_state.l) ? current_state.c + 1 : 0
```

**Защита от дрейфа:** если `|l_new - wall_clock| > <!-- parameters: ledger.hlc_max_drift_ms --> 60000ms` — выбрасывается `ClockDriftError`. Защищает от сломанного NTP.

### Idempotency

Каждый мутирующий запрос к Party содержит `idempotency_key` (UUID v4, глобально уникальный). Уникальность пары `(entity_id, idempotency_key)` гарантирует, что повторная отправка запроса не создаст дубликат версии.

Контракт: `idempotency_key ≡ version_uuid` — один ключ, одна версия.

### Retention

- Записи в ledger **не удаляются физически** (hard-delete запрещён).
- Физическое удаление (мягкое) — через `is_tombstone = true`.
- Фоновый compaction удаляет tombstones старше <!-- parameters: ledger.tombstone_retention_days --> 30 дней (90 дней для cross-DC).
- Party никогда не удаляется физически из системы — только мягкое удаление через `CacheStatus = inactive` или `is_tombstone = true`.
