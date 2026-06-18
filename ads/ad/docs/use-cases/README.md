---
version: "1.0.0"
updated: "2026-05-24"
agent: "architect"
status: draft
generated-from:
  - template: "templates-feat/160-use-case-map/use-case-map-template.md"
    feature: "use-case-map"
    agent: "architect"
    at: "2026-05-24"
    patch: false
---

# Use Case Map: ad

**Сервис:** ads
**Связанные документы:** Business Viewpoint, Domain Model, Security Policy, MVP Scope

---

## Operation Catalog

| Операция | Тип | Синхронность | Источник | Статус |
|----------|-----|-------------|----------|--------|
| CreateAd | command | sync | US-001, US-008 | draft |
| UpdateAd | command | sync | Domain Model (Structural Invariants — title, description, attributes) | draft |
| PublishAd | command | sync | Domain Model (State Machine — draft → active) | draft |
| ArchiveAd | command | sync | Domain Model (State Machine — active → archived) | draft |
| RestoreAd | command | sync | Domain Model (State Machine — archived → active) | draft |
| BlockAd | command | sync | BR-008, Security Policy (модерация) | draft |
| UnblockAd | command | sync | BR-008 | draft |
| SearchAds | query | sync | US-006, BR-007 | draft |
| GetAdById | query | sync | US-005, US-007 | draft |
| GetAdsFeed | query | sync | US-011 | draft |

### Пояснения к операциям

| Операция | Описание |
|----------|----------|
| **CreateAd** | Создание объявления (draft или active). Пользователь авторизован, party_id извлекается из JWT. Заполняются direction, category_id, title, description, attributes. Источники: US-001 (размещение объявления о продаже), US-008 (работа с черновиком) |
| **UpdateAd** | Обновление содержимого объявления (title, description, attributes). Доступно для draft и active. В archived — только после восстановления. Заблокированное (blocked=true) объявление можно редактировать, но изменение blocked запрещено (Structural Invariant). Источник: Domain Model |
| **PublishAd** | Публикация черновика: draft → active. Guard: все обязательные поля заполнены (category_id, title); blocked=false (если blocked=true — публикация заблокирована). Источник: Domain Model (State Machine) |
| **ArchiveAd** | Перевод active → archived. Пользователь подтвердил действие. Сотрудник — только своё объявление; администратор — любое. Источник: Domain Model (State Machine) |
| **RestoreAd** | Восстановление archived → active. Пользователь подтвердил действие. Сотрудник — только своё объявление; администратор — любое. Флаг blocked сохраняется. Источник: Domain Model (State Machine) |
| **BlockAd** | Установка флага блокировки на active-объявлении (blocked=true). Инициируется модератором или системой. Указывается причина блокировки. Применимо только к статусу active. Источник: BR-008, Security Policy (бизнес-угрозы — спам, мошенничество) |
| **UnblockAd** | Снятие флага блокировки (blocked → false). Только модератор/система. Если основной статус — active, объявление возвращается в поиск и матчинг. Источник: BR-008 |
| **SearchAds** | Глобальный поиск объявлений с фильтрацией. Результат: только status=active AND blocked=false. Фильтры: category_id, direction, party_id, title (полнотекстовый), динамические атрибуты. Поиск поддерживает курсорную пагинацию (keyset). Источник: US-006 (поиск и фильтрация), BR-007 (видимость: только active + не заблокированные) |
| **GetAdById** | Просмотр карточки объявления по ID. Права доступа: гость — только active; сотрудник предприятия-владельца — все статусы своего предприятия; администратор — любой. Источник: US-005 (просмотр детальной информации), US-007 (просмотр списка объявлений предприятия) |
| **GetAdsFeed** | Лента новых объявлений с фильтром direction=buy. Для поставщиков — видеть новые заявки на закупку. Результат: status=active AND blocked=false AND direction=buy, сортировка по created_at DESC. Поддерживает курсорную пагинацию. Источник: US-011 |

---

## Приоритет разработки

**MVP-приоритет (первые 5 операций для детальной разработки use case):**

1. **CreateAd** — core-операция, без которой нельзя разместить объявление
2. **UpdateAd** — создание и редактирование идут парой; пользователь должен иметь возможность исправить объявление
3. **SearchAds** — глобальный поиск с фильтрацией; ключевой сценарий для контрагентов
4. **GetAdById** — просмотр карточки объявления; необходимо для отображения детальной информации
5. **PublishAd** — публикация черновика; завершающий шаг в сценарии размещения

**Пост-MVP (следующие операции):**

6. ArchiveAd — управление жизненным циклом (устаревшие объявления)
7. RestoreAd — восстановление из архива
8. BlockAd — модерация объявлений
9. UnblockAd — разблокировка после устранения нарушений
10. GetAdsFeed — лента новых заявок на закупку

---

## Статусная модель операций

Все операции имеют статус `draft` — use case не написан. По мере детальной разработки use case статус будет повышен до `specified`, затем до `implemented`.
