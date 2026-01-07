Got it 👍
Here is a **clean, professional English documentation** that:

1. **Explains the legacy behavior first**
2. **Then explains exactly what we changed**
3. **Clarifies what did NOT change (ISO-legacy guarantees)**
4. Is suitable for **architecture review, audit, or handover**

You can paste this directly into **Confluence / README / design doc**.

---

# 📘 KOP CSV Processing – Legacy Behavior and New Configuration-Based Approach

## 1. Legacy Processing (Before Changes)

### 1.1 Overview

Historically, the KOP CSV packaging logic was entirely implemented in Java code.
All client-specific behaviors were **hardcoded** using nested `if/else` conditions.

The processing logic was mainly driven by:

* File path inspection
* Filename token matching
* Date detection
* Presence or absence of specific keywords (`IM`, `VM`, `FUTOPT`, `Interest`, etc.)

---

### 1.2 Legacy File Eligibility Logic

A file was considered eligible if:

* The file path contained the processing date (`yyyyMMdd` or `Date.toString()`)
* The file path did **not** contain `"writing"`
* Optional filters:

  * `report` token (must be present if defined)
  * `excludeReport` token (must NOT be present if defined)

Typical legacy logic:

```java
path.contains(date)
&& !path.contains("writing")
&& (report == null || path.contains(report))
&& (excludeReport == null || !path.contains(excludeReport))
```

---

### 1.3 Legacy File Renaming & Copying

For specific report types (e.g. `Account_COB`, `Journal_COB`, `Trade_COB`, `Interest`):

* Files were **copied**
* A **new filename** was generated based on:

  * Client code
  * Product family (e.g. FUTOPT)
  * Product type (`IM` / `VM` when present)
  * Report type
  * Processing date

Example legacy output:

```
CODECLIE--A811--BNPUS_VM_FUTOPT-CSV-20251017000000-Trade_COB-20251017.csv
```

---

### 1.4 ZIP Handling in Legacy

* ZIP files found during the scan were treated as **regular files**
* ZIP contents were **not scanned**
* ZIP files could be directly added to the final ZIP if eligible

---

### 1.5 Limitations of the Legacy Approach

* ❌ Adding a new client required Java code changes
* ❌ High risk of regression
* ❌ Very complex and hard-to-read logic
* ❌ Business rules tightly coupled to Java implementation
* ❌ Difficult to test and maintain

---

## 2. New Configuration-Based Approach (After Changes)

### 2.1 Objectives

The new approach was introduced to:

* Externalize client-specific business rules
* Avoid Java code changes when adding new clients
* Preserve **strict ISO-legacy behavior**
* Improve readability and maintainability
* Reduce operational and regression risks

---

## 2.2 Key Design Principles

* **Generic Java engine**
* **Client behavior described via YAML**
* **No regex complexity**
* **No dynamic templating**
* **Fallback to legacy behavior when needed**

---

## 2.3 Configuration Files

### Location

Client configuration files are now stored under:

```
src/main/resources/clients/
```

Examples:

```
clients/
 ├─ BNPUS.yml
 ├─ BNPSECETD.yml
 ├─ GRIKK.yml
 └─ default.yml
```

This ensures:

* No dependency on environment variables
* Identical behavior in DEV / UAT / PROD
* Compliance with security constraints

---

### 2.4 YAML Configuration Structure (Basic)

```yaml
clientCode: BNPSECETD
namingStrategy: DEFAULT

rules:
  - family: FUTOPT
    reportToken: INTEREST
    targetToken: INTEREST

  - family: FUTOPT
    reportToken: JOURNAL_COB
    targetToken: JOURNAL_COB
```

#### Field Meaning

| Field            | Description                       |
| ---------------- | --------------------------------- |
| `clientCode`     | Client identifier                 |
| `namingStrategy` | Java naming strategy to apply     |
| `family`         | Product family (e.g. FUTOPT)      |
| `reportToken`    | Token detected in source filename |
| `targetToken`    | Token used in generated filename  |

> YAML defines **what** to match, Java defines **how** to process.

---

## 2.5 Client Configuration Resolution

The configuration loading logic works as follows:

1. If a `clientCode` is provided:

   * Load `clients/<CLIENT>.yml`
2. If not found:

   * Load `clients/default.yml`
3. If no configuration is found:

   * **Fallback to legacy behavior**

➡️ This guarantees **no runtime failure** and **full backward compatibility**

---

## 2.6 File Processing Flow (New)

1. Recursive filesystem scan
2. Legacy eligibility checks (unchanged)
3. Special legacy case:

   * `report == null && excludeReport == null`
   * ZIP and FUTOPT CSVs copied as-is
4. Optional rule matching via YAML
5. Conditional file renaming and copying
6. ZIP entry path generation (unchanged)

---

## 2.7 Handling Files Without IM / VM (Interest Case)

### Issue Identified

Some reports (e.g. `Interest`) do not contain `IM` or `VM`.

Initial refactoring caused duplicated client codes:

```
BNPSECETD_BNPSECETD_FUTOPT-CSV-...
```

---

### Final Behavior (Corrected)

* Product type (`IM` / `VM`) is **optional**
* Included **only if detected**
* Otherwise omitted

Result:

```
BNPSECETD_FUTOPT-CSV-...
```

This behavior is **identical to legacy**.

---

## 2.8 Case Sensitivity

* YAML tokens (`reportToken`) are defined in **UPPERCASE**
* Java matching is **case-insensitive**

```java
path.toUpperCase().contains(reportToken)
```

This avoids regressions caused by inconsistent casing in source systems.

---

## 2.9 ZIP File Handling (Unchanged)

* ZIP files are treated as atomic files
* ZIP contents are never scanned
* ZIPs can be copied directly to the final archive

---

## 3. What Did NOT Change

✔ Eligibility logic
✔ Date detection logic
✔ ZIP behavior
✔ Legacy fallback scenarios
✔ Output format compatibility

---

## 4. Benefits of the New Approach

* ✅ New client = new YAML file only
* ✅ No Java rebuild required
* ✅ Easier audits and reviews
* ✅ Strong backward compatibility
* ✅ Cleaner, testable, maintainable codebase

---

## 5. Conclusion

The new configuration-based approach modernizes the KOP CSV processing while **fully preserving legacy behavior**.

It separates:

* **Business rules** (YAML)
* **Technical processing** (Java)

This provides a robust, scalable, and secure foundation for future client onboarding.

---

If you want, I can also provide:

* a **one-page executive summary**
* a **legacy vs new comparison table**
* a **flow diagram**
* or a **production validation checklist**

Just tell me 👍
