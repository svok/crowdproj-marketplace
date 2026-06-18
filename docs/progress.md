# Project Progress

**Общий прогресс:** ██░░░░░░░░░░░░░░░░░░ 11% (29/262)

---

## 📦 Бизнес-документация

### global
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 005 | Project Context | product-owner | ✅ | — | — |
| 010 | Lean Canvas | product-owner | ✅ | — | project-context |
| 020 | Value Proposition | product-owner | ✅ | — | lean-canvas |
| 030 | Personas | product-owner | ✅ | — | lean-canvas, value-proposition |
| 040 | Customer Journey Map | product-owner | ✅ | — | personas |
| 050 | TAM SAM SOM | product-owner | ✅ | — | lean-canvas |
| 060 | Monetization | product-owner | ✅ | — | business-model |
| 070 | Business Model | product-owner | ✅ | — | lean-canvas |
| 075 | Stakeholders | product-owner | ✅ | — | project-context, lean-canvas, business-model, personas |
| 080 | Premortem | product-owner | ✅ | — | lean-canvas, stakeholders |
| 090 | MVP Scope | product-owner | ✅ | — | user-scenarios |
| 100 | Business Requirements | product-owner | ✅ | — | lean-canvas, personas, business-model |

## 🔧 Доменная модель

### global
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 110 | Global Domain Rules | product-owner | ✅ | — | br |
| 120 | User Scenarios | product-owner | ✅ | — | personas, cjm, br |
| 130 | Service/Entity Decomposition | architect | ✅ | — | global-domain-rules, user-scenarios |
| 175 | Security Policy | product-owner | ✅ | — | stakeholders, global-domain-rules |
| 190 | Domain Flows | architect | ✅ | — | service-entity-decomposition, tech-flow |

### parties

#### party
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 140 | Domain Model | architect | ✅ | — | service-entity-decomposition |
| 150 | Entity Business Viewpoint | product-owner | ✅ | — | domain-model |
| 160 | Use Case Map | architect | 🔄 | — | entity-business-viewpoint |

##### RegisterParty
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ✅ | — | use-case-map |
| 180 | Technical Flow | architect | ✅ | — | use-case |
| 181 | Screen Flows | designer | 🔄 | — | use-case |
| 182 | Screen Specification | designer | ✅ | — | screen-flows |
| 185 | Test Specification | architect | ✅ | — | use-case, domain-model, tech-flow, security-policy |

##### UpdateParty
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ✅ | — | use-case-map |
| 180 | Technical Flow | architect | 🔄 | — | use-case |
| 181 | Screen Flows | designer | ✅ | — | use-case |
| 182 | Screen Specification | designer | 🔄 | — | screen-flows |
| 185 | Test Specification | architect | 🔄 | — | use-case, domain-model, tech-flow, security-policy |

##### SearchParties
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### GetPartyById
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### SetPartyStatus
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### CreateFictitiousParty
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### ClaimFictitiousParty
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

### ads

#### ad
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 140 | Domain Model | architect | ✅ | — | service-entity-decomposition |
| 150 | Entity Business Viewpoint | product-owner | 🔄 | — | domain-model |
| 160 | Use Case Map | architect | 🔄 | — | entity-business-viewpoint |

##### CreateAd
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### UpdateAd
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### PublishAd
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### ArchiveAd
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### RestoreAd
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### BlockAd
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### UnblockAd
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### SearchAds
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### GetAdById
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### GetAdsFeed
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

#### category
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 140 | Domain Model | architect | ✅ | — | service-entity-decomposition |
| 150 | Entity Business Viewpoint | product-owner | ✅ | — | domain-model |
| 160 | Use Case Map | architect | ✅ | — | entity-business-viewpoint |

##### GetCategoryTree
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### CreateCategory
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### UpdateCategory
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### ArchiveCategory
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### ActivateCategory
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

### matches

#### match
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 140 | Domain Model | architect | ⬜ | — | service-entity-decomposition |
| 150 | Entity Business Viewpoint | product-owner | ⬜ | — | domain-model |
| 160 | Use Case Map | architect | 🔄 | — | entity-business-viewpoint |

##### GetMatchesForAd
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### GenerateMatch
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### ExpireMatch
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

### notifications

#### notification
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 140 | Domain Model | architect | ⬜ | — | service-entity-decomposition |
| 150 | Entity Business Viewpoint | product-owner | ⬜ | — | domain-model |
| 160 | Use Case Map | architect | 🔄 | — | entity-business-viewpoint |

##### ScheduleNotification
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### ProcessNotification
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

##### UpdateLastVisit
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 170 | Use Case | architect | ⬜ | — | use-case-map |
| 180 | Technical Flow | architect | ⬜ | — | use-case |
| 181 | Screen Flows | designer | ⬜ | — | use-case |
| 182 | Screen Specification | designer | ⬜ | — | screen-flows |
| 185 | Test Specification | architect | ⬜ | — | use-case, domain-model, tech-flow, security-policy |

### ids (external)
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 191 | Domain Mapping & ACL | architect | ⬜ | integration-contract | integration-contract |
| 192 | State & Data Sync | architect | ⬜ | domain-mapping-acl | domain-mapping-acl |
| 193 | Business Fallback & Degradation | architect | ⬜ | integration-contract | integration-contract |

### notification-sender (external)
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 191 | Domain Mapping & ACL | architect | ⬜ | integration-contract | integration-contract |
| 192 | State & Data Sync | architect | ⬜ | domain-mapping-acl | domain-mapping-acl |
| 193 | Business Fallback & Degradation | architect | ⬜ | integration-contract | integration-contract |

## 🏗️ Архитектура

### global
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 200 | C4 System Context | architect | ⬜ | — | domain-flows |
| 210 | C4 Container | architect | ⬜ | c4-context (⬜) | c4-context |
| 230 | Tech Stack | architect | ⬜ | — | project-context, stakeholders |
| 235 | Security Architecture | architect | ⬜ | c4-context (⬜) | security-policy, c4-context |
| 240 | API Architecture | architect | ⬜ | security (⬜) | domain-model, use-case, security-policy, security |
| 250 | API Specification | architect | ⬜ | api-architecture (⬜) | api-architecture, use-case |
| 260 | Architecture Decision Record | architect | ⬜ | — | — |

### parties

| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 220 | C4-3 Service | architect | ⬜ | c4-container (⬜) | c4-container |

#### party
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 222 | C4-3 Entity | architect | ⬜ | — | domain-model, tech-flow |

### ads

| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 220 | C4-3 Service | architect | ⬜ | c4-container (⬜) | c4-container |

#### ad
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 222 | C4-3 Entity | architect | ⬜ | — | domain-model, tech-flow |

#### category
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 222 | C4-3 Entity | architect | ⬜ | — | domain-model, tech-flow |

### matches

| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 220 | C4-3 Service | architect | ⬜ | c4-container (⬜) | c4-container |

#### match
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 222 | C4-3 Entity | architect | ⬜ | — | domain-model, tech-flow |

### notifications

| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 220 | C4-3 Service | architect | ⬜ | c4-container (⬜) | c4-container |

#### notification
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 222 | C4-3 Entity | architect | ⬜ | — | domain-model, tech-flow |

### ids (external)
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 261 | Integration Contract | architect | ⬜ | — | service-entity-decomposition |
| 262 | Security Boundary | architect | ⬜ | integration-contract | integration-contract |
| 263 | Limits & Cost Modeling | architect | ⬜ | integration-contract | integration-contract |

### notification-sender (external)
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 261 | Integration Contract | architect | ⬜ | — | service-entity-decomposition |
| 262 | Security Boundary | architect | ⬜ | integration-contract | integration-contract |
| 263 | Limits & Cost Modeling | architect | ⬜ | integration-contract | integration-contract |

## ⚙️ Разработка

### global
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 400 | Project Initialization | architect | ⬜ | tech-stack (⬜) | tech-stack |
| 415 | API Specification Generator | architect | ⬜ | api-spec (⬜) | api-spec |
| 540 | SQLx4K Repository (PostgreSQL) | architect | 🔶 | repo-common | repo-common |

### parties

| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 410 | App Ktor | architect | ⬜ | project | project |

#### party
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 420 | API Models | architect | ⬜ | api-spec (⬜) | api-spec |
| 440 | API Mappers | architect | ⬜ | api-models (⬜), internal-models | api-models, internal-models |
| 450 | Stubs & Factories | architect | ⬜ | — | test-spec |
| 460 | Entity App Ktor | architect | ⬜ | — | tech-flow |
| 470 | Business Logic Skeleton | architect | ⬜ | — | use-case |
| 480 | Business Rule Router | architect | ⬜ | — | global-domain-rules, use-case |
| 483 | Business Rule Stub | architect | ⬜ | stubs (⬜), biz-skeleton (⬜) | stubs, biz-skeleton |
| 492 | Business Rule Auth Sketch | architect | ⬜ | security (⬜), biz-rule (⬜) | security, biz-rule |
| 495 | Business Rule Custom | architect | ⬜ | biz-rule (⬜) | biz-rule |
| 498 | Business Logic Pass | architect | ⬜ | biz-rule (⬜) | tech-flow, biz-rule |
| 501 | Business Flow Integration Tests | architect | ⬜ | biz-logic (⬜), security (⬜) | biz-logic, tech-flow, test-spec, security |

### ads

| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 410 | App Ktor | architect | ⬜ | project | project |

#### ad
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 420 | API Models | architect | ⬜ | api-spec (⬜) | api-spec |
| 440 | API Mappers | architect | ⬜ | api-models (⬜), internal-models | api-models, internal-models |
| 450 | Stubs & Factories | architect | ⬜ | — | test-spec |
| 460 | Entity App Ktor | architect | ⬜ | — | tech-flow |
| 470 | Business Logic Skeleton | architect | ⬜ | — | use-case |
| 480 | Business Rule Router | architect | ⬜ | — | global-domain-rules, use-case |
| 483 | Business Rule Stub | architect | ⬜ | stubs (⬜), biz-skeleton (⬜) | stubs, biz-skeleton |
| 492 | Business Rule Auth Sketch | architect | ⬜ | security (⬜), biz-rule (⬜) | security, biz-rule |
| 495 | Business Rule Custom | architect | ⬜ | biz-rule (⬜) | biz-rule |
| 498 | Business Logic Pass | architect | ⬜ | biz-rule (⬜) | tech-flow, biz-rule |
| 501 | Business Flow Integration Tests | architect | ⬜ | biz-logic (⬜), security (⬜) | biz-logic, tech-flow, test-spec, security |

#### category
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 420 | API Models | architect | ⬜ | api-spec (⬜) | api-spec |
| 440 | API Mappers | architect | ⬜ | api-models (⬜), internal-models | api-models, internal-models |
| 450 | Stubs & Factories | architect | ⬜ | — | test-spec |
| 460 | Entity App Ktor | architect | ⬜ | — | tech-flow |
| 470 | Business Logic Skeleton | architect | ⬜ | — | use-case |
| 480 | Business Rule Router | architect | ⬜ | — | global-domain-rules, use-case |
| 483 | Business Rule Stub | architect | ⬜ | stubs (⬜), biz-skeleton (⬜) | stubs, biz-skeleton |
| 492 | Business Rule Auth Sketch | architect | ⬜ | security (⬜), biz-rule (⬜) | security, biz-rule |
| 495 | Business Rule Custom | architect | ⬜ | biz-rule (⬜) | biz-rule |
| 498 | Business Logic Pass | architect | ⬜ | biz-rule (⬜) | tech-flow, biz-rule |
| 501 | Business Flow Integration Tests | architect | ⬜ | biz-logic (⬜), security (⬜) | biz-logic, tech-flow, test-spec, security |

### matches

| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 410 | App Ktor | architect | ⬜ | project | project |

#### match
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 420 | API Models | architect | ⬜ | api-spec (⬜) | api-spec |
| 440 | API Mappers | architect | ⬜ | api-models (⬜), internal-models | api-models, internal-models |
| 450 | Stubs & Factories | architect | ⬜ | — | test-spec |
| 460 | Entity App Ktor | architect | ⬜ | — | tech-flow |
| 470 | Business Logic Skeleton | architect | ⬜ | — | use-case |
| 480 | Business Rule Router | architect | ⬜ | — | global-domain-rules, use-case |
| 483 | Business Rule Stub | architect | ⬜ | stubs (⬜), biz-skeleton (⬜) | stubs, biz-skeleton |
| 492 | Business Rule Auth Sketch | architect | ⬜ | security (⬜), biz-rule (⬜) | security, biz-rule |
| 495 | Business Rule Custom | architect | ⬜ | biz-rule (⬜) | biz-rule |
| 498 | Business Logic Pass | architect | ⬜ | biz-rule (⬜) | tech-flow, biz-rule |
| 501 | Business Flow Integration Tests | architect | ⬜ | biz-logic (⬜), security (⬜) | biz-logic, tech-flow, test-spec, security |

### notifications

| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 410 | App Ktor | architect | ⬜ | project | project |

#### notification
| № | Задача | Агент | Статус | Конфликт | Зависит от |
| --- | --- | --- | --- | --- | --- |
| 420 | API Models | architect | ⬜ | api-spec (⬜) | api-spec |
| 440 | API Mappers | architect | ⬜ | api-models (⬜), internal-models | api-models, internal-models |
| 450 | Stubs & Factories | architect | ⬜ | — | test-spec |
| 460 | Entity App Ktor | architect | ⬜ | — | tech-flow |
| 470 | Business Logic Skeleton | architect | ⬜ | — | use-case |
| 480 | Business Rule Router | architect | ⬜ | — | global-domain-rules, use-case |
| 483 | Business Rule Stub | architect | ⬜ | stubs (⬜), biz-skeleton (⬜) | stubs, biz-skeleton |
| 492 | Business Rule Auth Sketch | architect | ⬜ | security (⬜), biz-rule (⬜) | security, biz-rule |
| 495 | Business Rule Custom | architect | ⬜ | biz-rule (⬜) | biz-rule |
| 498 | Business Logic Pass | architect | ⬜ | biz-rule (⬜) | tech-flow, biz-rule |
| 501 | Business Flow Integration Tests | architect | ⬜ | biz-logic (⬜), security (⬜) | biz-logic, tech-flow, test-spec, security |
