---
version: "1.0.4"
updated: "2026-05-24"
generated-from:
  - template: "templates-feat/140-domain-model/domain-model-template.md"
    feature: "domain-model"
    agent: "architect"
    at: "2026-05-17"
    patch: false
  - template: "templates-feat/170-use-case/use-case-template.md"
    feature: "use-case"
    agent: "architect"
    at: "2026-05-17"
    patch: true
    reason: "Cross-doc conflict resolution with parties/user/docs/use-cases/uc-user-deactivate-cache.md — added missing IdS status event trigger (enabled/disabled/deleted)"
---
# Domain Model: user
**Связанные документы:** Decomposition, Global Domain Rules, BR-001, BR-002
---
## Aggregate: user
Корень агрегата кэша данных пользователя в сервисе parties. Source-of-Truth — IdS (Keycloak/Casdoor). Parties хранит локальный кэш данных пользователя, необходимых для отображения на UI (карточки товаров, профили предприятий). Управление жизненным циклом пользователя (создание, приглашение, роли, аутентификация) полностью во внешнем IdS — parties не содержит бизнес-логики управления пользователями.
**Идентификатор:** UserId (UUID из IdS)
**Тип:** Aggregate Root
**Хранение:** state-sourced
---
## Value Objects
Значимые типы, идентифицируемые по значению. Каждый — с правилами валидации.

| VO | Тип | Правила валидации |
|----|-----|-------------------|
| UserId | UUID (string) | Уникальный идентификатор из IdS, соответствует формату RFC 4122 |
| PartyId | UUID (string) | Ссылка на Aggregate Root Party (parties/party). Не null. Связь один-ко-многим: один Party → много User |
| DisplayName | string | Имя пользователя для отображения на UI. Не пустой, длина 1–200 символов |
| Email | string | RFC 5322, не пустой. **Не уникален в parties** — один email может быть привязан к нескольким PartyId (сотрудник нескольких предприятий согласно BR-002) |
| CacheStatus | enum | Статус кэша: `active` | `inactive`. Отражает актуальность данных из IdS. Причины перехода между статусами — вне контекста parties (определяются IdS) |

---
## Cache Status (состояние кэша)
User в parties — это кэш данных из IdS, не управляемая сущность. Вместо State Machine (не применима — нет бизнес-логики переходов) используется простой статус кэша:

| Статус | Описание |
|--------|----------|
| active | Данные актуальны, пользователь отображается в UI (карточки товаров, профили предприятий) |
| inactive | Пользователь удалён/отключён/деактивирован в IdS. Не отображается в публичных UI-компонентах |
> **Решение:** parties не реализует State Machine для user. Логика переходов между статусами — ответственность IdS. Parties получает конечный статус из IdS и хранит только `active`/`inactive`.
**Контракт маппинга статусов IdS → CacheStatus (user):**
| Статус в IdS | CacheStatus в parties | Условие |
|--------------|----------------------|---------|
| active | active | Пользователь активен в IdS, кэш актуален для отображения в публичных UI |
| invited | inactive | Пользователь приглашён, но ещё не активировал учётную запись — не отображается в публичных UI до установки пароля |
| disabled, deleted | inactive | Пользователь деактивирован/удалён — не показывать в UI |
> Полный жизненный цикл статусов пользователя (invited → active → disabled) — ответственность IdS. Parties является потребителем конечного состояния.
---
## Structural Invariants
Правила, истинные ВСЕГДА для агрегата, в любом состоянии.

| Инвариант | Комментарий |
|-----------|-------------|
| UserId != null | Каждый user в parties имеет идентификатор из IdS |
| PartyId != null | Каждый user привязан к предприятию (Party) |
| Email соответствует RFC 5322 | Валидация формата email |
| DisplayName не пустой (1–200 символов) | Имя для отображения обязательно |
---
## Domain Events
События, которые публикует агрегат parties/user.

| Событие | Когда публикуется | Данные |
|---------|------------------|--------|
| UserCacheUpdated | При обновлении кэша данных пользователя из IdS (создание по событию UserRegisteredWithGroup, изменение профиля, смена статуса) | userId, partyId, cacheStatus, updatedAt |
**Триггеры (входящие события от IdS):**
- `OrganizationCreated` — создание предприятия → создание Party + привязка Admin User
- `UserRegisteredWithGroup` — добавление сотрудника → создание/обновление кэша User
- Обновление профиля пользователя в IdS — обновление displayName/email в кэше
- Статусное событие из IdS (enabled/disabled/deleted) — обновление cacheStatus в кэше User
