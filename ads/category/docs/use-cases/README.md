---
version: "1.0.0"
updated: "2026-05-24"
agent: "architect"
status: approved
generated-from:
  - template: "templates-feat/160-use-case-map/use-case-map-template.md"
    feature: "use-case-map"
    agent: "architect"
    at: "2026-05-24"
    patch: false
---

# Use Case Map: category

**Связанные документы:** Business Viewpoint, Domain Model, BR-003, BR-007, Decomposition

---

## Operation Catalog

| Операция | Тип | Синхронность | Источник | Статус |
|----------|-----|-------------|----------|--------|
| GetCategoryTree | query | sync | BR-003 §26 (иерархическое дерево категорий), BR-007 §116 (дерево категорий как фильтр поиска) | draft |
| CreateCategory | command | sync | BR-003 §26 (иерархическое дерево категорий — администратор создаёт узлы и подузлы) | draft |
| UpdateCategory | command | sync | Domain Model — обновление name, attribute_definitions, sort_order | draft |
| ArchiveCategory | command | sync | Domain Model — переход Active→Archived или Draft→Archived | draft |
| ActivateCategory | command | sync | Domain Model — переход Draft→Active (guard: attribute_definitions заполнен) | draft |

---

## Приоритет разработки

**Выбранный приоритет:** GetCategoryTree, CreateCategory, UpdateCategory
