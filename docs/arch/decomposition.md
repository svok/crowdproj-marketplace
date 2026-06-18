---
version: "2.0.7"
updated: "2026-06-05"
agent: "architect"
status: approved
decomposition:
  bounded_contexts:
    - name: parties
      aggregate_roots:
        - name: party
          service: parties
          internal_entities: [user]
          value_objects: [companyName, inn, email]
          enums: "cache_status: ACTIVE/INACTIVE"
      encapsulated_services: [parties]
    - name: ads
      aggregate_roots:
        - name: ad
          service: ads
          internal_entities: []
          value_objects: [productAttributes]
          enums: "direction: SELL/BUY; status: DRAFT/ACTIVE/ARCHIVED/BLOCKED"
        - name: category
          service: ads
          internal_entities: []
          value_objects: [categoryPath, attributeSchema]
          enums: ""
      encapsulated_services: [ads]
    - name: matches
      aggregate_roots:
        - name: match
          service: matches
          internal_entities: []
          value_objects: []
          enums: "lifecycle: ACTIVE/EXPIRED; stage: INTEREST/TRIAL_ORDER/DEAL_CLOSED"
      encapsulated_services: [matches]
    - name: notifications
      aggregate_roots:
        - name: notification
          service: notifications
          internal_entities: []
          value_objects: [notificationContent]
          enums: "channel: EMAIL; status: PENDING/SENT/FAILED"
      encapsulated_services: [notifications]
  blackboxes:
    - name: ids
      display_name: "Identity Service"
      product: "Keycloak"
      role: "OIDC auth, user management, org management"
      port_interface: "IdentityProvider"
      integration_pattern: "ACL"
      related_service: "parties"
    - name: notification-sender
      display_name: "Notification Delivery Gateway"
      product: "Notification Delivery Gateway"
      role: "Delivery of notifications via providers"
      port_interface: "AsyncDeliveryPort"
      integration_pattern: "ACL"
      related_service: "notifications"
generated-from:
  - template: "templates-feat/130-service-entity-decomposition/decomposition-template.md"
    feature: "service-entity-decomposition"
    agent: "architect"
    at: "2026-06-04"
    patch: false
    reason: "Initial creation"
  - template: "templates-feat/130-service-entity-decomposition/decomposition-template.md"
    feature: "service-entity-decomposition"
    agent: "architect"
    at: "2026-06-04"
    patch: true
    reason: "Reentry: полная переработка — notifications-split, external systems, post-match tracking, contact-proxy"
  - template: "templates-feat/130-service-entity-decomposition/decomposition-template.md"
    feature: "service-entity-decomposition"
    agent: "architect"
    at: "2026-06-04"
    patch: true
    reason: "Reentry: приведение к формату шаблона (агрегаты, полный YAML frontmatter, бизнесовый уровень)"
  - template: "templates-feat/130-service-entity-decomposition/decomposition-template.md"
    feature: "service-entity-decomposition"
    agent: "architect"
    at: "2026-06-05"
    patch: true
    reason: "Reentry: финализация — приведение к шаблону, исправление frontmatter, уточнение SoT (party→parties, user→IdS), добавление Bounded Contexts, добавление notification-sender в YAML"
  - template: "templates-feat/130-service-entity-decomposition/decomposition-template.md"
    feature: "service-entity-decomposition"
    agent: "architect"
    at: "2026-06-05"
    patch: true
    reason: "PO review: fix enums consistency (ad_status, match_status, notification_status), remove 'дайджесты' per GDR, add out-of-scope note, add async-matcher SLA note"
  - template: "templates-feat/130-service-entity-decomposition/decomposition-template.md"
    feature: "service-entity-decomposition"
    agent: "architect"
    at: "2026-06-05"
    patch: true
    reason: "PO rejection fixes: fill YAML value_objects+enums per table, merge user into party (internal entity), lowercase BC names, unify external system names (ids), replace _event-bus_ with (broadcast)"
---

# Service/Entity Decomposition — CrowdProj Marketplace

**Связанные документы:** Global Domain Rules, User Scenarios

---

## 1. Агрегаты и их состав

| Агрегат (Корень) | Сервис-владелец ({srv}) | Корень агрегата ({root-entity}) | Внутренние сущности ({internal-entity}) | Value Objects ({value-object}) | Ключевые атрибуты и Enum |
|------------------|------------------------|-------------------------------|----------------------------------------|-------------------------------|--------------------------|
| party | parties | party | user | companyName, inn, email | cacheStatus: ACTIVE/INACTIVE (DISABLED/DELETED мапятся в INACTIVE) |
| ad | ads | ad | — | productAttributes | direction: SELL/BUY; status: DRAFT/ACTIVE/ARCHIVED/BLOCKED |
| category | ads | category | — | categoryPath, attributeSchema | — |
| match | matches | match | — | — | lifecycle: ACTIVE/EXPIRED; stage: INTEREST/TRIAL_ORDER/DEAL_CLOSED |
| notification | notifications | notification | — | notificationContent | channel: EMAIL; status: PENDING/SENT/FAILED |

---

## 2. Bounded Contexts (группировка агрегатов)

| Bounded Context | Агрегаты в контексте | Бизнес-назначение |
|-----------------|---------------------|-------------------|
| parties | party | Управление контрагентами (Party — интегральная таблица организаций, user — внутренняя сущность-кэш из IdS) |
| ads | ad, category | Управление объявлениями и иерархией категорий |
| matches | match | Матчинг объявлений sell↔buy и трекинг пост-матчинговых взаимодействий |
| notifications | notification | Подготовка и агрегация уведомлений |

---

## Сервисы ({srv})

| Сервис ({srv}) | Bounded Context | Назначение | Ответственность |
|----------------|-----------------|------------|-----------------|
| parties | parties | Справочник контрагентов (ЮЛ/ИП/самозанятые) | Party (ИНН, регион, сфера деятельности). Реестр предприятий (поиск + фильтр). Внутренняя сущность user (кэш из IdS). Contact-proxy (форма связи без раскрытия ПД). Управление пользователями — во внешнем IdS. |
| ads | ads | Управление объявлениями (продажа/закупка) | CRUD объявлений. Иерархия категорий. Поиск по объявлениям. Блокировка/модерация. |
| matches | matches | Матчинг объявлений sell↔buy | Генерация связок при создании/обновлении ad. Post-match tracking (статусы взаимодействия: интерес, пробный заказ, завершение сделки). |
| notifications | notifications | Подготовка уведомлений | Одно письмо за цикл неактивности. Выбор канала. Форматирование под канал. Пользовательские настройки уведомлений (preferences). Приоритеты. Хранение last_visit_at. Cron поиска «молчунов». |

---

## Внешние и Blackbox-системы (External Services)

| Система ({external_svc}) | Продукт | Роль в предметной области | Паттерн интеграции | Порт-интерфейс | SLA / Лимиты | Covered Scenarios | Covered Operations | Связан с сервисом |
|---------------------------|---------|--------------------------|-------------------|----------------|-------------|-------------------|--------------------|-------------------|
| ids (IdS / Keycloak-Casdoor) | IdS (Keycloak/Casdoor) | Аутентификация, управление пользователями и организациями | ACL | Admin API / Events | — | US-001, US-008, US-018, US-019 | User CRUD, аутентификация, роли, приглашения, эмиссия событий | parties |
| notification-sender (Шлюз доставки уведомлений) | Шлюз доставки уведомлений | Доставка уведомлений через провайдеров | ACL | Async API | — | US-007, US-013 | Отправка через провайдера, повторные попытки, сбор статусов доставки | notifications |

---

## Ownership & Source-of-Truth

| Сущность | Source-of-Truth (сервис) | Реплицируется в |
|----------|--------------------------|-----------------|
| party | parties | — |
| user | IdS | parties (кэш: userId, partyId, displayName, email, cacheStatus) |
| ad | ads | — |
| category | ads | — |
| match | matches | — |
| notification | notifications | — |

---

## Границы взаимодействия

| От (сервис) | К (сервис) | Тип | Описание |
|-------------|-----------|-----|----------|
| ads | matches | async | AdCreated / AdUpdated — генерация/перегенерация связок |
| matches | notifications | async | MatchFound — подготовка уведомления |
| ids | notifications | async | UserLoggedIn — обновление last_visit_at |
| notifications | ids | sync | Запрос контактных данных получателя в момент подготовки уведомления |
| ids | parties | async | OrganizationCreated / UserRegisteredWithGroup — создание кэша. Статусные события (enabled/disabled/deleted) |
| notifications | notification-sender | async | Передача подготовленного уведомления |
| notification-sender | notifications | async | Статус доставки |
| parties | notifications | async | ContactRequest — запрос связи с контрагентом (уведомление владельца объявления) |
| parties | (broadcast) | async | UserCacheUpdated — для будущих потребителей |
| matches | (broadcast) | async | MatchInteractionRecorded — статус взаимодействия после матча |

---

## Примечания

**Async-матчинг и SLA:** Взаимодействие `ads → matches` помечено как `async`, но бизнес-сценарии US-003/US-010 требуют показа матчей «сразу после создания» объявления. Это означает, что async-доставка должна обеспечивать near-real-time задержку (SLA < 1 с). Для MVP допустимо синхронное создание матчей в рамках обработки события (фаза 2: фронтенд запрашивает matches синхронно после успешного создания объявления — см. Domain Flows §Cross-Service Orchestration).

**Out of scope (MVP):** Billing/подписки и Аналитика спроса (упомянутые в Lean Canvas) — находятся вне границ текущей декомпозиции. В MVP отсутствуют как отдельные сервисы/агрегаты. Будут добавлены на следующих этапах развития платформы.
