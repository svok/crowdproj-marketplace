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

# Use Case: ReactivateUserCacheFromIdS

**Entity:** user
**Связанные документы:** Domain Model, Business Viewpoint, Use Case Map

---

## Preconditions

Что должно быть истинно ДО вызова операции.

| # | Предусловие | Тип проверки |
|---|-------------|-------------|
| 1 | В parties существует user с указанным UserId (кэш из IdS) | State — кэш user существует |
| 2 | CacheStatus user = inactive (кэш неактивен, ожидает реактивации) | State — статус кэша inactive |
| 3 | Idempotency-ключ не был обработан ранее (уникальность в рамках операции) | Invariant — уникальность idempotency key |

---

## Postconditions

Что гарантированно истинно ПОСЛЕ успешного выполнения.

| # | Постусловие | Тип результата |
|---|-------------|---------------|
| 1 | CacheStatus user = active | State — новый статус кэша |
| 2 | Поле updatedAt = текущее время | Field — изменённое поле |
| 3 | Событие UserCacheUpdated опубликовано с cacheStatus=active | Event — опубликованное событие |

---

## Основной поток (Happy Path)

Пошаговая последовательность действий системы.

1. Система получает асинхронное статусное событие от IdS (enabled/unblocked) для UserId
2. Система проверяет idempotency-ключ события — не был ли он уже обработан
3. Система загружает User по UserId из хранилища
4. Система проверяет, что CacheStatus user = inactive (если уже active — идемпотентный пропуск)
5. Система устанавливает CacheStatus = active и updatedAt = текущее время
6. Система сохраняет User (INSERT в ledger, новая версия)
7. Система публикует событие UserCacheUpdated (userId, partyId, cacheStatus=active, updatedAt) через outbox

---

## Альтернативные потоки

Все отклонения от happy path.

| # | Ситуация | Шаг | Действие системы |
|---|----------|-----|-----------------|
| 1 | Idempotency-ключ уже был обработан (дубликат события) | 2 | Пропуск операции — событие уже обработано. Операция завершена без ошибки |
| 2 | User с указанным UserId не найден в parties | 3 | Логирование ошибки, отправка события в DLQ для ручного разбора. Возможна гонка (событие создания ещё не получено, или запись удалена) |
| 3 | CacheStatus user уже active | 4 | Пропуск шагов 5–7 — операция успешно завершена (идемпотентность), событие UserCacheUpdated не публикуется повторно |

---

## Системные ограничения

Нефункциональные параметры операции.

| Ограничение | Значение |
|-------------|----------|
| Sync/Async | async — команда по статусному событию из IdS (event bus) |
| Таймаут | <!-- parameters: parties.user.async_command_timeout_minutes --> 5 минут |
| Идемпотентность | Да — по idempotency key (повторное событие enabled/unblocked для того же UserId безопасно) |
| Транзакционность | В рамках одного агрегата — только parties/user (cacheStatus, updatedAt, INSERT в ledger) |
