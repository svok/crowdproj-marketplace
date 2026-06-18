---
version: "1.0.2"
updated: "2026-05-24"
agent: "architect"
status: draft
generated-from:
  - template: "templates-feat/160-use-case-map/use-case-map-template.md"
    feature: "use-case-map"
    agent: "architect"
    at: "2026-05-22"
    patch: false
  - template: "templates-feat/160-use-case-map/use-case-map-template.md"
    feature: "use-case-map"
    agent: "designer"
    at: "2026-05-22"
    patch: true
    reason: "UX review: added visibility matrix by status (active/inactive/fictitious) for SearchParties and GetPartyById. Added note about dynamic field set (mandatory + DynamicAttributes) from IdS."
  - template: "templates-feat/160-use-case-map/use-case-map-template.md"
    feature: "use-case-map"
    agent: "architect"
    at: "2026-05-22"
    patch: true
    reason: "Review fix: removed redundant DeactivateParty (covered by SetPartyStatus). Added Outbound Events section. Clarified ClaimFictitiousParty sync/async duality."
  - template: "templates-feat/160-use-case-map/use-case-map-template.md"
    feature: "use-case-map"
    agent: "architect"
    at: "2026-05-24"
    patch: true
    reason: "Reentry fix: фактическое удаление DeactivateParty из каталога, маппинга событий и приоритета (предыдущий патч декларировал удаление, но строки остались)."
---

# Use Case Map: party

**Связанные документы:** Business Viewpoint, Domain Model

**Сервис:** parties

---

## Operation Catalog

| Операция | Тип | Синхронность | Источник | Статус |
|----------|-----|-------------|----------|--------|
| RegisterParty | command | async (IdS — внешняя система) | US-001, US-008 | draft |
| UpdateParty | command | async (IdS — внешняя система) | Domain Model — обновление кэша из IdS | draft |
| SearchParties | query | sync | US-004, US-012 | draft |
| GetPartyById | query | sync | US-005, US-015 | draft |
| SetPartyStatus | command | async (IdS — внешняя система) | Domain Model — active↔inactive | draft |
| CreateFictitiousParty | command | async (краулер — фоновая задача) | Domain Model — PartyFictitiousCreated (MVP+) | draft |
| ClaimFictitiousParty | command | sync | Domain Model — fictitious→active (MVP+) | draft |

### Примечание: динамический состав полей

Party в parties — кэш данных из IdS. Состав полей организации **не фиксирован**: IdS может передавать разный набор атрибутов в зависимости от настройки профиля организации. Parties хранит:
- **Mandatory (жёсткая схема):** PartyId, Inn, PartyName, RegistrationType, CacheStatus
- **DynamicAttributes (semi-structured):** остальные поля (контакты, сфера деятельности, регион, ОГРН, согласия и т.д.) — как `jsonb`/динамическая карта

Операции `RegisterParty` и `UpdateParty` принимают от IdS динамический набор полей без жёсткой фиксации схемы. Валидация формата — ответственность IdS. Parties проверяет только mandatory-поля и формат Inn.

### Маппинг на события-триггеры

| Операция | Триггер / Входящее событие |
|----------|---------------------------|
| RegisterParty | `OrganizationCreated` из IdS |
| UpdateParty | Обновление профиля организации в IdS |
| SetPartyStatus | Статусное событие из IdS (enabled/disabled/blocked/deleted) |
| CreateFictitiousParty | Внутренняя команда краулера (MVP+) |
| ClaimFictitiousParty | **Двустадийно:** (1) async — событие `OrganizationCreated` из IdS с совпадающим ИНН; (2) sync — подтверждение полномочий пользователем (UI-шаг) |

---

### Особенности видимости по статусам

Операции `SearchParties` и `GetPartyById` учитывают статус организации:

| Статус | Видимость в SearchParties | Видимость в GetPartyById | Пометка в UI |
|--------|--------------------------|--------------------------|-------------|
| active | Да (публичный реестр) | Да (стандартная карточка) | — |
| inactive | По умолчанию скрыт. Доступен при явной фильтрации `status=inactive` (исторические данные) | По умолчанию скрыт. Доступен владельцу или при явном запросе с подтверждением | «Организация отключена» |
| fictitious (MVP+) | Да (реестр) | Да | «Неподтверждённая организация» |

Поле `CacheStatus` (или `RegistrationType`) определяет пометку в UI. Фильтрация по статусу — параметр запроса `status` в `SearchParties`.

---

### Outbound Events

| Операция | Публикуемое событие | Описание |
|----------|-------------------|----------|
| RegisterParty | `PartyCacheCreated` | Создан кэш Party после получения OrganizationCreated из IdS |
| UpdateParty | `PartyCacheUpdated` | Обновлены кэшированные данные предприятия из IdS |
| SetPartyStatus | `PartyCacheStatusChanged` | Изменён статус кэша (active↔inactive) по событию из IdS |
| CreateFictitiousParty | `PartyFictitiousCreated` | Создана фиктивная запись краулером (MVP+) |
| ClaimFictitiousParty | `PartyClaimed` | Фиктивная запись сопоставлена с реальной регистрацией в IdS (MVP+) |

---

### Расшифровка синхронности

- **sync** — вызывающая сторона ждёт ответа (REST запрос, пользователь в UI ждёт). Операция выполняется быстрее <!-- parameters: api.async_threshold_ms --> 500ms.
- **async** — вызывающая сторона не ждёт (постановка в очередь, фоновый процесс, обработка события). Операция зависит от внешней системы (IdS) или фонового процесса (краулер), либо может выполняться дольше <!-- parameters: api.async_threshold_ms --> 500ms.

---

## Приоритет разработки

**Выбранный приоритет (MVP):**
1. **RegisterParty** — создание учётной записи организации (ядро two-sided marketplace)
2. **UpdateParty** — обновление данных организации (из IdS)
3. **SearchParties** — поиск по реестру предприятий
4. **GetPartyById** — просмотр карточки предприятия
5. **SetPartyStatus** — управление активностью организации (active↔inactive)

**MVP+ (пост-MVP):**
- CreateFictitiousParty — создание фиктивной записи краулером
- ClaimFictitiousParty — сопоставление фиктивной записи с реальной регистрацией
