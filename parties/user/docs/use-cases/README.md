---
version: "1.0.4"
updated: "2026-05-24"
agent: "architect"
status: approved
generated-from:
  - template: "templates-feat/160-use-case-map/use-case-map-template.md"
    feature: "use-case-map"
    agent: "architect"
    at: "2026-05-17"
    patch: false
  - template: "templates-feat/160-use-case-map/use-case-map-template.md"
    feature: "use-case-map"
    agent: "architect"
    at: "2026-05-24"
    patch: true
---
# Use Case Map: user
**Связанные документы:** Business Viewpoint, Domain Model
---
## Operation Catalog
| Операция | Тип | Синхронность | Источник | Статус |
|----------|-----|-------------|----------|--------|
| GetUserById | query | sync | US-004, US-005, US-014 | draft |
| GetUsersByParty | query | sync | US-018 | draft |
| SearchUsers | query | sync | US-018 (поиск сотрудников предприятия) | draft |
| SyncUserCacheFromIdS | command | async | Domain Model — триггеры: `OrganizationCreated`, `UserRegisteredWithGroup`, обновление профиля в IdS | draft |
| DeactivateUserCacheFromIdS | command | async | Domain Model — триггер: disabled/deleted из IdS | draft |
| RegisterUserFromOrganization | command | async | Domain Model — триггер: `OrganizationCreated` (привязка Admin User); tech-flow RegisterParty §113-122 | draft |
| ReactivateUserCacheFromIdS | command | async | Domain Model — триггер: enabled/unblocked из IdS (восстановление cacheStatus в active) | draft |
---
## Приоритет разработки
**Выбранный приоритет:** 1) SyncUserCacheFromIdS, 2) RegisterUserFromOrganization, 3) DeactivateUserCacheFromIdS, 4) ReactivateUserCacheFromIdS, 5) GetUserById, 6) GetUsersByParty, 7) SearchUsers
---
## Outbound Events
| Событие | Инициатор | Данные | Назначение |
|---------|-----------|--------|------------|
| UserCacheUpdated | SyncUserCacheFromIdS, RegisterUserFromOrganization, DeactivateUserCacheFromIdS, ReactivateUserCacheFromIdS | userId, partyId, cacheStatus, updatedAt | Оповещение смежных сервисов об изменении кэша пользователя
