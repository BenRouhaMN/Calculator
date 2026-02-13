Parfait.
Voici la **V2.1 complète consolidée**, fusionnée avec la V2 précédente + gestion de cohérence en cascade + contraintes renforcées + logique clearing réelle.

Structure prête pour comité d’architecture.

---

# 🔷 1️⃣ PARTY – V2.1

### Table: PARTY

| Attribute   | Definition                          | Type (Java)   | Constraint                        |
| ----------- | ----------------------------------- | ------------- | --------------------------------- |
| partyId     | Technical unique identifier         | Long          | PK, not null                      |
| partyName   | Official legal or natural name      | String        | Not null                          |
| partyType   | LEGAL_PARTY / NATURAL_PERSON        | String (ENUM) | Not null                          |
| legalForm   | SA, PLC, SARL…                      | String        | Nullable                          |
| countryCode | ISO-3166 alpha-2                    | String        | Length = 2, not null              |
| lei         | Legal Entity Identifier (ISO 17442) | String        | Unique when not null, length = 20 |
| status      | ACTIVE / INACTIVE                   | String (ENUM) | Not null                          |
| validFrom   | Legal validity start date           | LocalDate     | Not null                          |
| validTo     | Legal validity end date             | LocalDate     | Nullable                          |
| createdAt   | Creation timestamp                  | LocalDateTime | Not null                          |
| updatedAt   | Last update timestamp               | LocalDateTime | Not null                          |

---

### Additional Context / Constraints

* LEI must be unique when present.
* Only one ACTIVE Party per LEI at a given date.
* Soft delete only (status = INACTIVE + validTo set).
* validFrom ≤ validTo (if validTo not null).
* Party cannot be INACTIVE while active roles exist.
* If Party becomes INACTIVE → all related PartyRoles must be end-dated.
* Cascade closure must generate AuditEvent entries.

---

# 🔷 2️⃣ PARTY_ROLE – V2.1

### Table: PARTY_ROLE

| Attribute              | Definition                                                                                  | Type          | Constraint   |
| ---------------------- | ------------------------------------------------------------------------------------------- | ------------- | ------------ |
| partyRoleId            | Technical identifier                                                                        | Long          | PK           |
| partyId                | Reference to PARTY                                                                          | Long          | FK, not null |
| roleType               | CLIENT / CLEARING_FIRM / BOOKING_FIRM / EXECUTING_FIRM / EXCHANGE / CLEARING_HOUSE / BROKER | String (ENUM) | Not null     |
| regulatoryJurisdiction | ISO-3166 country code                                                                       | String        | Nullable     |
| startDate              | Role validity start                                                                         | LocalDate     | Not null     |
| endDate                | Role validity end                                                                           | LocalDate     | Nullable     |
| status                 | ACTIVE / INACTIVE                                                                           | String (ENUM) | Not null     |
| createdAt              | Creation timestamp                                                                          | LocalDateTime | Not null     |
| updatedAt              | Last update timestamp                                                                       | LocalDateTime | Not null     |

---

### Additional Context / Constraints

* A Party may have multiple roles.
* Only one active identical role per jurisdiction at a given date.
* Role validity must fall within Party validity.
* No active Role if Party is INACTIVE.
* If PartyRole becomes INACTIVE or expires → all active Relationships referencing it must be end-dated.
* No Relationship can exist with an INACTIVE PartyRole.

---

# 🔷 3️⃣ RELATIONSHIP_RULE – V2.1

### Table: RELATIONSHIP_RULE

| Attribute            | Definition                                                                                                    | Type          | Constraint |
| -------------------- | ------------------------------------------------------------------------------------------------------------- | ------------- | ---------- |
| ruleId               | Technical identifier                                                                                          | Long          | PK         |
| relationshipType     | CLEARS_THROUGH / OPERATES_THROUGH / EXECUTES_THROUGH / BOOKS_THROUGH / CLEARING_MEMBER_OF / TRADING_MEMBER_OF | String (ENUM) | Not null   |
| relationshipCategory | HIERARCHY / ROLE                                                                                              | String (ENUM) | Not null   |
| fromRoleType         | Origin role type                                                                                              | String (ENUM) | Not null   |
| toRoleType           | Target role type                                                                                              | String (ENUM) | Not null   |
| maxActiveParents     | Max active parents per scope                                                                                  | Integer       | ≥ 1        |
| allowOverlap         | Allow overlapping validity                                                                                    | Boolean       | Not null   |
| description          | Business rule description                                                                                     | String        | Not null   |
| createdAt            | Timestamp                                                                                                     | LocalDateTime | Not null   |
| updatedAt            | Timestamp                                                                                                     | LocalDateTime | Not null   |

---

### Additional Context / Constraints

* Enforced at Relationship creation AND update.
* maxActiveParents applied per (scopeType + scopeCode).
* If allowOverlap = false → no overlapping validity periods for same scope.
* Used for structural validation only (not business context).

---

# 🔷 4️⃣ RELATIONSHIP – V2.1 (Core Engine)

### Table: RELATIONSHIP

| Attribute          | Definition             | Type          | Constraint                     |
| ------------------ | ---------------------- | ------------- | ------------------------------ |
| relationshipId     | Technical identifier   | Long          | PK                             |
| fromPartyRoleId    | Origin role            | Long          | FK PARTY_ROLE, not null        |
| toPartyRoleId      | Target role            | Long          | FK PARTY_ROLE, not null        |
| relationshipRuleId | Governing rule         | Long          | FK RELATIONSHIP_RULE, not null |
| scopeType          | GLOBAL / MARKET / CCP  | String (ENUM) | Not null                       |
| scopeCode          | Identifier of scope    | String        | Nullable if GLOBAL             |
| startDate          | Validity start         | LocalDate     | Not null                       |
| endDate            | Validity end           | LocalDate     | Nullable                       |
| status             | ACTIVE / INACTIVE      | String (ENUM) | Not null                       |
| comment            | Business justification | String        | Nullable                       |
| createdAt          | Creation timestamp     | LocalDateTime | Not null                       |
| updatedAt          | Update timestamp       | LocalDateTime | Not null                       |

---

## 🔒 Critical Constraints

### Scope logic

* If scopeType = GLOBAL → scopeCode must be NULL.
* If scopeType ≠ GLOBAL → scopeCode must be NOT NULL.

### Uniqueness constraint

Unique combination:

```
(fromPartyRoleId,
 toPartyRoleId,
 relationshipRuleId,
 scopeType,
 scopeCode,
 startDate)
```

### Temporal integrity

* Relationship validity must be within validity of both PartyRoles.
* No active relationship allowed with inactive role.
* Relationship must respect allowOverlap from rule.
* maxActiveParents enforced per (scopeType + scopeCode).

### Cascade rule

If PartyRole becomes inactive or end-dated:
→ All related active Relationships must be end-dated.
→ AuditEvent must be generated with changedBy = SYSTEM.

---

# 🔷 5️⃣ AUDIT_EVENT – V2.1

### Table: AUDIT_EVENT

| Attribute    | Definition                                | Type          | Constraint |
| ------------ | ----------------------------------------- | ------------- | ---------- |
| auditId      | Technical identifier                      | Long          | PK         |
| entityType   | PARTY / PARTY_ROLE / RELATIONSHIP         | String (ENUM) | Not null   |
| entityId     | Impacted entity id                        | Long          | Not null   |
| action       | CREATE / UPDATE / DELETE / CASCADE_UPDATE | String (ENUM) | Not null   |
| oldValue     | JSON snapshot                             | String        | Nullable   |
| newValue     | JSON snapshot                             | String        | Nullable   |
| changedBy    | User or SYSTEM                            | String        | Not null   |
| changeReason | Business justification                    | String        | Nullable   |
| changeDate   | Timestamp                                 | LocalDateTime | Not null   |

---

### Additional Context

* Immutable (no updates allowed).
* One row per mutation.
* Cascade operations must explicitly generate entries.

---

# 🔥 What This V2.1 Now Supports

✔ Multi-market clearing
✔ Multi-CCP clearing
✔ Subclearing chains
✔ Temporal consistency
✔ Controlled overlaps
✔ Structural validation via rules
✔ Cascade integrity
✔ Full audit traceability
✔ Regulatory reporting readiness

---

# 🔷 ENTITY ROLE SUMMARY (for committee)

---

## PARTY

Represents a legal or natural entity participating in the ecosystem.
Acts as the master data anchor for all roles and relationships.

---

## PARTY_ROLE

Represents a functional capacity assumed by a Party (Client, Clearing Firm, Exchange…).
Defines how a Party participates in the derivatives ecosystem.

---

## RELATIONSHIP_RULE

Defines allowed structural interactions between role types.
Enforces cardinality, overlap, and integrity constraints.

---

## RELATIONSHIP

Represents a real operational or structural interaction between two roles.
Scope-aware (GLOBAL / MARKET / CCP), time-bound, and rule-governed.

---

## AUDIT_EVENT

Ensures immutable traceability of all data mutations.
Supports regulatory compliance and forensic analysis.

---

# 🎯 Architecture Verdict

This V2.1 is:

* Structurally robust
* Clearing-ready
* Subclearing-compatible
* Governance-safe
* Production-grade

---

Si tu veux maintenant je peux :

* Te faire la version "one-slide architecture summary"
* Simuler un cas complet ENGIE → KOP → GSS → ICE CCP
* Ou identifier les risques restants avant comité

Dis-moi ton prochain besoin.

