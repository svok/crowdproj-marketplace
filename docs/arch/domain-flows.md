---
version: "1.0.4"
updated: "2026-06-05"
generated-from:
  - template: "templates-feat/190-domain-flows/domain-flows-template.md"
    feature: "domain-flows"
    agent: "architect"
    at: "2026-05-17T00:00:00Z"
    patch: false
  - template: "templates-feat/140-domain-model/domain-model-template.md"
    feature: "domain-model"
    agent: "architect"
    at: "2026-05-17"
    patch: true
    reason: "Cross-doc conflict resolution with parties/user/docs/domain-model.md — added UserCacheUpdated event, updated decomposition dependency to v1.0.5"
  - template: "templates-feat/170-use-case/use-case-template.md"
    feature: "use-case"
    agent: "architect"
    at: "2026-05-17"
    patch: true
    reason: "Cross-doc conflict resolution with parties/user/docs/use-cases/uc-user-deactivate-cache.md — added IdS→parties status events boundary (enabled/disabled/deleted)"
  - template: "templates-feat/130-service-entity-decomposition/decomposition-template.md"
    feature: "service-entity-decomposition"
    agent: "architect"
    at: "2026-06-05"
    patch: true
    reason: "Cross-doc sync with decomposition v2.0.7: Party SoT → parties, add missing events (ContactRequest, UserCacheUpdated→broadcast, MatchInteractionRecorded→broadcast)"
---
# Domain Flows — CrowdProj Marketplace
**Связанные документы:** Decomposition, Domain Rules, User Scenarios
---
## Cross-Service Orchestration
Процессы, затрагивающие более одного сервиса. Тип: `orchestration` — координатор + компенсации; `choreography` — события без координатора.
| Flow | Участники (сервисы) | Тип | Описание |
|------|---------------------|-----|----------|
| Публикация объявления | ads, matches, notifications | choreography | **Фаза 1 (изменение — choreography):** При создании/обновлении объявления (ads) отправляется событие AdCreated/AdUpdated → matches получает событие, генерирует/перегенерирует связки sell↔buy (match-таблица) и отправляет MatchFound → notifications планирует email-уведомление. **Фаза 2 (чтение — Query):** Фронтенд после успешного создания/обновления объявления сразу запрашивает matches (синхронно) для немедленного отображения встречных объявлений пользователю. Sync-запрос не управляет процессом публикации, а лишь отображает его текущий результат |
| Регистрация предприятия | IdS (Keycloak/Casdoor), parties | choreography | **Контракт фронтенда:** при регистрации предприятия фронтенд обязан передавать ИНН и ОГРН как атрибуты (Group Attributes) создаваемой организации через OIDC Admin API. IdS включает ИНН/ОГРН в payload события OrganizationCreated. IdS **нативно** отправляет email-приглашение на указанный адрес (через встроенный механизм Keycloak/Casdoor). **Затем** parties получает событие, извлекает ИНН/ОГРН из атрибутов события и создаёт кэш Party. Source-of-Truth для Party: **parties** (нативное хранение). Идентификационные данные Party могут поступать из разных источников: через события IdS при регистрации (OrganizationCreated), а также через создание фиктивных записей (LLM-генерация в MVP+). IdS управляет пользователями (user), parties управляет организациями (party). Подробнее — см. `parties/party/docs/domain-model.md` |
| Приглашение сотрудника | IdS (Keycloak/Casdoor), parties | choreography | Администратор приглашает сотрудника по email → во фронтенде через IdS Admin API создаётся User в существующей Organization. IdS **нативно** отправляет письмо-приглашение со ссылкой для установки пароля. IdS эмитит UserRegisteredWithGroup (с существующим organization_id). Parties получает событие и связывает user с существующей Party (без создания нового Party) |
| Вход пользователя | IdS (Keycloak/Casdoor), notifications | choreography | IdS эмитит UserLoggedIn при каждом успешном логине или обновлении токена → notifications обновляет `last_visit_at` локально (без batch-запросов к IdS) |
| Email-напоминание о матчингах | notifications, IdS | choreography | Cron-задача в notifications: local SELECT по своей БД для поиска «молчунов» (сравнение `last_visit_at` с настроенным периодом). Точечный sync-запрос к IdS (GET /userinfo) за email строго в момент отправки письма. Отправка одного письма о новых матчингах за цикл неактивности |
---
## Intra-Service Queries (Read-Only)
Внутрисервисные запросы чтения, не требующие межсервисного взаимодействия. Каждый запрос выполняется автономно внутри одного сервиса — его Source-of-Truth.
| Query | Сервис | Тип | Описание |
|-------|--------|-----|----------|
| Поиск по реестру предприятий | parties | sync (HTTP) | Полнотекстовый поиск и фильтрация контрагентов по региону / сфере деятельности (US-004, BR-005) |
| Глобальный поиск по объявлениям | ads | sync (HTTP) | Поиск товаров с фильтрацией по категориям, регионам и JSONB-характеристикам (US-006, BR-007) |
| Лента новых заявок | ads | sync (HTTP) | Вывод актуальных объявлений на закупку (direction=buy) для поставщиков (US-011) |
---
## Sagas & Compensations
Все flows в MVP имеют тип `choreography` — сервисы реагируют на события без центрального координатора. Процессы, требующие компенсаций (orchestration), в MVP отсутствуют.
| Saga (= Flow) | Шаги | Компенсация при ошибке шага |
|---------------|------|---------------------------|
| — | — | Нет orchestration-процессов в MVP. Все flows — choreography, компенсации не требуются |
---
## Async Domain Interactions
Все асинхронные связи между сервисами. Собраны из Decomposition (Границы взаимодействия).
| Отправитель ({srv}) | Получатель ({srv}) | Событие | Механизм |
|---------------------|--------------------|---------|----------|
| ads | matches | AdCreated, AdUpdated | Kafka (event bus) |
| matches | notifications | MatchFound | Kafka (event bus) |
| IdS (Keycloak/Casdoor) | notifications | UserLoggedIn | Kafka (event bus) |
| IdS (Keycloak/Casdoor) | parties | OrganizationCreated (новая организация), UserRegisteredWithGroup (новый пользователь в существующей организации) | Kafka (event bus) |
| IdS (Keycloak/Casdoor) | parties | Статусные события пользователя (enabled/disabled/deleted) — обновление cacheStatus в parties/user | Kafka (event bus) |
| parties | (broadcast) | UserCacheUpdated (обновление локального кэша пользователя: создание, смена статуса, обновление профиля) | Kafka (event bus) — для будущих потребителей (notifications, ads) |
| parties | notifications | ContactRequest (запрос связи с контрагентом — уведомление владельца объявления) | Kafka (event bus) |
| matches | (broadcast) | MatchInteractionRecorded (статус взаимодействия после матча) | Kafka (event bus)
