---
version: "1.0.0"
updated: "2026-05-24"
agent: "architect"
status: draft
generated-from:
  - template: "templates-feat/170-use-case/use-case-template.md"
    feature: "use-case"
    agent: "architect"
    at: "2026-05-24"
    patch: false
---

# Use Case: RegisterUserFromOrganization

**Entity:** user
**Связанные документы:** Domain Model, Business Viewpoint, Use Case Map

---

## Preconditions

Что должно быть истинно ДО вызова операции. Проверяет parties при получении события OrganizationCreated из IdS.

| # | Предусловие | Тип проверки |
|---|-------------|-------------|
| 1 | Party с указанным PartyId (OrganizationId из OrganizationCreated) уже создана в parties/party | State — существование зависимой сущности |
| 2 | User с указанным UserId не существует в parties/user (первый пользователь — администратор предприятия) | State — уникальность агрегата |
| 3 | Idempotency-ключ события OrganizationCreated не обработан ранее | Idempotency — защита от дубликатов |

---

## Postconditions

Что гарантированно истинно ПОСЛЕ успешного выполнения.

| # | Постусловие | Тип результата |
|---|-------------|---------------|
| 1 | User создан с CacheStatus=active | State — новый статус кэша |
| 2 | Поля User: UserId, PartyId, DisplayName, Email, CacheStatus=active, updatedAt заполнены | Field — созданные поля |
| 3 | Событие UserCacheUpdated опубликовано с userId, partyId, cacheStatus=active, updatedAt | Event — опубликованное событие |

---

## Основной поток (Happy Path)

Пошаговая последовательность. Каждый шаг — одно атомарное действие системы.

1. Система получает событие `OrganizationCreated` от IdS (async — через брокер сообщений)
2. Система проверяет idempotency-ключ события — если ключ уже обработан, пропускает (игнорирует дубликат)
3. Система загружает Party по PartyId (OrganizationId из события) для валидации существования зависимой записи
4. Система проверяет, что User с указанным UserId не существует в parties/user
5. Система создаёт User: UserId, PartyId, DisplayName, Email (данные из OrganizationCreated), CacheStatus=active
6. Система сохраняет User в parties/user с HLC (hlc_l, hlc_c, hlc_instance_id, hlc_dc_id) и version_uuid
7. Система записывает событие UserCacheUpdated в outbox-таблицу в рамках одной транзакции
8. Система публикует событие `UserCacheUpdated` из outbox-таблицы

---

## Альтернативные потоки

Все отклонения от happy path.

| # | Ситуация | Шаг | Действие системы |
|---|----------|-----|-----------------|
| 1 | Idempotency-ключ уже обработан (дубликат события) | 2 | Система игнорирует событие — User уже создан. Завершение без ошибки |
| 2 | Party с указанным PartyId не найдена в parties/party | 3 | Система помещает событие в DLQ (dead-letter queue). Требуется ручная синхронизация — gap между созданием Party и получением события |
| 3 | User с указанным UserId уже существует | 4 | Система возвращает ошибку DuplicateUser — «Пользователь с таким UserId уже зарегистрирован». Событие помечается как failed |
| 4 | Ошибка сохранения User (конкурентный доступ, сбой БД) | 6 | Система выполняет retry до 3 раз с экспоненциальной задержкой. После 3 неудач — ошибка ConcurrencyConflict, событие помещается в DLQ |
| 5 | Ошибка публикации события UserCacheUpdated из outbox | 7 | Outbox-запись остается в таблице со статусом pending. Фоновый процесс повторной обработки retry-ит публикацию |

---

## Системные ограничения

Нефункциональные параметры операции.

| Ограничение | Значение |
|-------------|----------|
| Sync/Async | async — операция запускается по событию OrganizationCreated из IdS (event bus) |
| Таймаут | <!-- parameters: parties.user.async_command_timeout_minutes --> 5 минут — максимальное время обработки события (включая retry) |
| Идемпотентность | Да — по idempotency-ключу в событии из IdS. Повторные события с тем же ключом игнорируются |
| Транзакционность | Outbox Pattern — сохранение User и запись в outbox в рамках одной БД-транзакции. HLC total order, optimistic locking |
