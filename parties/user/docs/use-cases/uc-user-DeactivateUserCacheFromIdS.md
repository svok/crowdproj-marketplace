---
version: "1.0.3"
updated: "2026-05-24"
status: draft
generated-from:
  - template: "templates-feat/170-use-case/use-case-template.md"
    feature: "use-case"
    agent: "architect"
    at: "2026-05-17"
    patch: false
  - template: "templates-feat/170-use-case/use-case-template.md"
    feature: "use-case"
    agent: "architect"
    at: "2026-05-24"
    patch: true
---
# Use Case: DeactivateUserCacheFromIdS
**Entity:** user
**Связанные документы:** Domain Model, Business Viewpoint, Use Case Map
---
## Preconditions
Что должно быть истинно ДО вызова операции.
| # | Предусловие | Тип проверки |
|---|-------------|-------------|
| 1 | В parties существует user с указанным UserId (кэш из IdS) | State — кэш user существует |
| 2 | Idempotency-ключ события от IdS не обработан ранее | Invariant — проверка идемпотентности |
| 3 | Событие из IdS: disabled или deleted для данного UserId | Auth — событие получено от доверенного источника (IdS) |
---
## Postconditions
Что гарантированно истинно ПОСЛЕ успешного выполнения.
| # | Постусловие | Тип результата |
|---|-------------|---------------|
| 1 | CacheStatus user = inactive | State — новый статус кэша |
| 2 | Поле updatedAt = текущее время | Field — изменённое поле |
| 3 | Событие UserCacheUpdated опубликовано с cacheStatus=inactive | Event — опубликованное событие |
---
## Основной поток (Happy Path)
Пошаговая последовательность действий системы.
1. Система получает асинхронное событие от IdS (disabled/deleted) для UserId
2. Система проверяет, что idempotency-ключ события не обработан ранее
3. Система загружает User по UserId из хранилища
4. Система проверяет текущий CacheStatus user
5. Система устанавливает CacheStatus = inactive и updatedAt = текущее время
6. Система сохраняет User (INSERT в ledger, новая версия)
7. Система публикует событие UserCacheUpdated (userId, partyId, cacheStatus=inactive, updatedAt)
---
## Альтернативные потоки
Все отклонения от happy path.
| # | Ситуация | Шаг | Действие системы |
|---|----------|-----|-----------------|
| 1 | User с указанным UserId не найден в parties | 3 | Логирование предупреждения — возможна гонка событий (событие создания еще не получено) или запись была удалена ранее. Операция завершена без ошибки |
| 2 | CacheStatus user уже inactive | 4 | Пропуск шагов 5-7 — операция успешно завершена (идемпотентная обработка), событие UserCacheUpdated не публикуется повторно |
---
## Системные ограничения
Нефункциональные параметры операции.
| Ограничение | Значение |
|-------------|----------|
| Sync/Async | async — команда по событию из IdS (event bus) |
| Таймаут | <!-- parameters: parties.user.async_command_timeout_minutes --> 5 минут |
| Идемпотентность | Да — повторное событие disabled/deleted для того же UserId безопасно: статус уже inactive, событие не дублируется |
| Транзакционность | В рамках одного агрегата — только parties/user (cacheStatus, updatedAt) |
