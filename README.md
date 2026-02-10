
Parfait.
Je reprends **exactement la proposition précédente**, **même format**, et j’ajoute **un exemple concret pour CHAQUE entité**, cohérent entre eux (même scénario de bout en bout).

👉 **Scénario fil rouge utilisé pour tous les exemples**

* **ENGIE SA** est **client**
* **BNP Paribas SA** est **clearing firm**
* Relation : **ENGIE (CLIENT) CLEARS_THROUGH BNP (CLEARING_FIRM)**
* Source initiale : **UBIX**

---

# 🟦 ENTITY — Entity data model

### Modèle

| Attribute         | Definition and possible values      | Type (Java)   | Constraint                    |
| ----------------- | ----------------------------------- | ------------- | ----------------------------- |
| entityId          | Technical unique identifier         | Long          | PK, not null                  |
| entityName        | Official legal or natural name      | String        | Not null                      |
| entityDescription | Free text description               | String        | Nullable                      |
| entityType        | LEGAL_ENTITY | NATURAL_PERSON       | String        | Not null                      |
| legalForm         | SA, PLC, GmbH, SARL…                | String        | Nullable                      |
| countryCode       | ISO-3166 alpha-2 country code       | String        | Length = 2, not null          |
| lei               | Legal Entity Identifier (ISO 17442) | String        | Unique, length = 20, nullable |
| status            | ACTIVE | INACTIVE                   | String        | Not null                      |
| validFrom         | Entity legal validity start date    | LocalDate     | Not null                      |
| validTo           | Entity legal validity end date      | LocalDate     | Not null                      |
| sourceSystem      | UBIX | MANUAL | EXTERNAL            | String        | Not null                      |
| createdAt         | Creation timestamp                  | LocalDateTime | Not null                      |
| updatedAt         | Last update timestamp               | LocalDateTime | Not null                      |
| version           | Optimistic locking version          | Integer       | Not null                      |

### Exemple

```
ENTITY
entityId        = 1001
entityName      = "ENGIE SA"
entityType      = LEGAL_ENTITY
legalForm       = "SA"
countryCode     = "FR"
lei             = "5493001KJTIIGC8Y1R12"
status          = ACTIVE
validFrom       = 2008-01-01
validTo         = 9999-12-31
sourceSystem    = "UBIX"
createdAt       = 2024-01-10T09:15:00
updatedAt       = 2024-01-10T09:15:00
version         = 1
```

---

# 🟦 ENTITY_ROLE — Entity Role data model

### Modèle

| Attribute              | Definition and possible values                                                               | Type (Java)   | Constraint            |
| ---------------------- | -------------------------------------------------------------------------------------------- | ------------- | --------------------- |
| entityRoleId           | Technical unique identifier                                                                  | Long          | PK, not null          |
| entityId               | Reference to ENTITY                                                                          | Long          | FK → ENTITY, not null |
| role                   | CLIENT | CLEARING_FIRM | BOOKING_FIRM | BRANCH | CCP | EXCHANGE | ASSET_MANAGER | FUND_ADMIN | String        | Not null              |
| roleScope              | GLOBAL | CLEARING | EXECUTION | COLLATERAL                                                   | String        | Not null              |
| regulatoryJurisdiction | ISO-3166 country code                                                                        | String        | Length = 2, nullable  |
| startDate              | Role validity start date                                                                     | LocalDate     | Not null              |
| endDate                | Role validity end date                                                                       | LocalDate     | Not null              |
| status                 | ACTIVE | INACTIVE                                                                            | String        | Not null              |
| sourceSystem           | UBIX | MANUAL | EXTERNAL                                                                     | String        | Not null              |
| createdAt              | Creation timestamp                                                                           | LocalDateTime | Not null              |
| updatedAt              | Last update timestamp                                                                        | LocalDateTime | Not null              |
| version                | Optimistic locking version                                                                   | Integer       | Not null              |

### Exemples

**ENGIE comme CLIENT**

```
ENTITY_ROLE
entityRoleId = 2001
entityId     = 1001   (ENGIE SA)
role         = "CLIENT"
roleScope    = "CLEARING"
startDate    = 2020-01-01
endDate      = 9999-12-31
status       = ACTIVE
sourceSystem = "UBIX"
version      = 1
```

**BNP comme CLEARING_FIRM**

```
ENTITY
entityId   = 1002
entityName = "BNP Paribas SA"
```

```
ENTITY_ROLE
entityRoleId = 2002
entityId     = 1002   (BNP Paribas SA)
role         = "CLEARING_FIRM"
roleScope    = "CLEARING"
startDate    = 2010-01-01
endDate      = 9999-12-31
status       = ACTIVE
sourceSystem = "UBIX"
version      = 1
```

---

# 🟦 RELATIONSHIP_RULE — Relationship Rule data model

### Modèle

| Attribute            | Definition and possible values | Type (Java)   | Constraint   |
| -------------------- | ------------------------------ | ------------- | ------------ |
| ruleId               | Technical unique identifier    | Long          | PK, not null |
| relationshipType     | Type of relationship governed  | String        | Not null     |
| relationshipCategory | ROLE | HIERARCHY               | String        | Not null     |
| fromRole             | Origin role                    | String        | Not null     |
| toRole               | Target role                    | String        | Not null     |
| maxActiveParents     | Max active parent roles        | Integer       | ≥ 1          |
| maxActiveChildren    | Max active child roles         | Integer       | ≥ 1          |
| allowOverlap         | Allow date overlap             | Boolean       | Not null     |
| requiresHierarchy    | Requires hierarchy validation  | Boolean       | Not null     |
| description          | Business rule description      | String        | Not null     |
| createdAt            | Creation timestamp             | LocalDateTime | Not null     |
| updatedAt            | Last update timestamp          | LocalDateTime | Not null     |
| version              | Optimistic locking version     | Integer       | Not null     |

### Exemple

```
RELATIONSHIP_RULE
ruleId               = 3001
relationshipType     = "CLEARS_THROUGH"
relationshipCategory = "ROLE"
fromRole             = "CLIENT"
toRole               = "CLEARING_FIRM"
maxActiveParents     = 1
maxActiveChildren    = 9999
allowOverlap         = false
requiresHierarchy    = false
description          = "A client can clear through only one active clearing firm at a given time"
version              = 1
```

---

# 🟦 RELATIONSHIP — Relationship data model

### Modèle

| Attribute            | Definition and possible values                                            | Type (Java)   | Constraint                       |
| -------------------- | ------------------------------------------------------------------------- | ------------- | -------------------------------- |
| relationshipId       | Technical unique identifier                                               | Long          | PK, not null                     |
| fromEntityRoleId     | Origin EntityRole                                                         | Long          | FK → ENTITY_ROLE, not null       |
| toEntityRoleId       | Target EntityRole                                                         | Long          | FK → ENTITY_ROLE, not null       |
| relationshipType     | CLEARS_THROUGH | OPERATES_THROUGH | EXECUTES_FOR | MEMBER_OF | MANAGED_BY | String        | Not null                         |
| relationshipCategory | ROLE | HIERARCHY                                                          | String        | Not null                         |
| relationshipRuleId   | Governing relationship rule                                               | Long          | FK → RELATIONSHIP_RULE, not null |
| priority             | Resolution priority (default = 1)                                         | Integer       | ≥ 1                              |
| startDate            | Relationship validity start date                                          | LocalDate     | Not null                         |
| endDate              | Relationship validity end date                                            | LocalDate     | Not null                         |
| status               | ACTIVE | INACTIVE                                                         | String        | Not null                         |
| comment              | Free text justification                                                   | String        | Nullable                         |
| sourceSystem         | UBIX | MANUAL | EXTERNAL                                                  | String        | Not null                         |
| createdAt            | Creation timestamp                                                        | LocalDateTime | Not null                         |
| updatedAt            | Last update timestamp                                                     | LocalDateTime | Not null                         |
| version              | Optimistic locking version                                                | Integer       | Not null                         |

### Exemple

```
RELATIONSHIP
relationshipId        = 4001
fromEntityRoleId      = 2001   (ENGIE / CLIENT)
toEntityRoleId        = 2002   (BNP / CLEARING_FIRM)
relationshipType      = "CLEARS_THROUGH"
relationshipCategory  = "ROLE"
relationshipRuleId    = 3001
startDate             = 2021-01-01
endDate               = 9999-12-31
status                = ACTIVE
comment               = "Standard clearing agreement"
sourceSystem          = "UBIX"
version               = 1
```

---

# 🟦 AUDIT_EVENT — Audit trail data model

### Modèle

| Attribute    | Definition and possible values             | Type (Java)   | Constraint   |
| ------------ | ------------------------------------------ | ------------- | ------------ |
| auditId      | Technical unique identifier                | Long          | PK, not null |
| entityType   | ENTITY | ENTITY_ROLE | RELATIONSHIP | RULE | String        | Not null     |
| entityId     | Identifier of impacted entity              | Long          | Not null     |
| action       | CREATE | UPDATE | DELETE                   | String        | Not null     |
| oldValue     | Previous state (JSON serialized)           | String        | Nullable     |
| newValue     | New state (JSON serialized)                | String        | Nullable     |
| changedBy    | User or system                             | String        | Not null     |
| changeReason | Business justification                     | String        | Nullable     |
| changeDate   | Change timestamp                           | LocalDateTime | Not null     |

### Exemple

```
AUDIT_EVENT
auditId      = 9001
entityType   = "RELATIONSHIP"
entityId     = 4001
action       = "CREATE"
oldValue     = null
newValue     = "{fromRole:ENGIE_CLIENT, toRole:BNP_CLEARING_FIRM}"
changedBy    = "caar-admin"
changeReason = "Initial migration from UBIX"
changeDate   = 2024-01-10T09:45:00
```

---

# 🧠 Cohérence globale (important)

* **ENTITY** = identité stable
* **ENTITY_ROLE** = comportement contextualisé
* **RELATIONSHIP_RULE** = garde-fou structurel
* **RELATIONSHIP** = lien effectif entre rôles
* **AUDIT_EVENT** = traçabilité complète

👉 Le modèle est **canonique**, **gouverné**, **audit-ready**, et **compatible sortie UBIX**.

---

