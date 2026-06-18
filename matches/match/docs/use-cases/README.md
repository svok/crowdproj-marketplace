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

# Use Case Map: match

**Связанные документы:** Business Viewpoint (BR-003, BR-004, BR-006, BR-008), Domain Model

---

## Operation Catalog

| Операция | Тип | Синхронность | Источник | Статус |
|----------|-----|-------------|----------|--------|
| GetMatchesForAd | query | sync | US-007, US-009 (BR-003 §US-1, BR-004 §US-1/2) | draft |
| GenerateMatch | command | async | Domain Model — генерация при AdCreated/AdUpdated | draft |
| ExpireMatch | command | async | Domain Model — перевод в expired при изменении/удалении объявления | draft |

---

## Приоритет разработки

**Выбранный приоритет:** GetMatchesForAd, GenerateMatch
