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

# Use Case Map: notification

**Связанные документы:** Business Viewpoint, Domain Model

---

## Operation Catalog

| Операция | Тип | Синхронность | Источник | Статус |
|----------|-----|-------------|----------|--------|
| ScheduleNotification | command | async | Domain Model — событие MatchFound от сервиса matches | draft |
| ProcessNotification | command | async | Domain Model — cron-триггер: local SELECT молчунов, sync email из IdS, отправка письма | draft |
| UpdateLastVisit | command | async | Domain Model — событие UserLoggedIn от IdS | draft |

---

## Приоритет разработки

**Выбранный приоритет:** ScheduleNotification → UpdateLastVisit → ProcessNotification
