# SQL Injection - Lab #17: SQL Injection with Filter Bypass via XML Encoding

**YouTube Tutorial:** [SQL Injection - Lab #17 SQL injection with filter bypass via XML encoding (Short Version)](https://youtu.be/ELdyZm0nK4g)

---

## 1. What is SQLi with WAF Filter Bypass via XML Encoding?

### Core Concept

Unlike the blind labs, this injection is **not blind** — the application returns the results of the SQL query directly in the HTTP response, so a **UNION-based attack** can retrieve data from other tables. The catch is a **Web Application Firewall (WAF)** that inspects the raw request and blocks any payload containing obvious SQL injection keywords (like `UNION`, `SELECT`, `FROM`) with an **"Attack detected"** response.

The bypass works because the vulnerable input sits **inside an XML document**. The WAF checks the request **before** the XML parser decodes it, so if the payload is obfuscated as **XML entities**, the raw bytes no longer match the WAF's keyword signatures:

* **Request as sent:** `<storeId>1 &#x55;NION &#x53;ELECT ...</storeId>` — no literal `UNION`/`SELECT` bytes, so the WAF lets it through.
* **Request as parsed:** The XML parser decodes `&#x55;` → `U`, `&#x53;` → `S`, etc., **before** the value is concatenated into the SQL query.
* **SQL as executed:** The backend receives the fully decoded `1 UNION SELECT username || '~' || password FROM users` and happily returns the credentials.

The WAF filters the **raw HTTP layer**; the application's XML parser reverses the obfuscation before the query ever hits the database — a classic encoding-context mismatch.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Endpoint:** `POST /product/stock` — the **stock check feature** on any product page.
* **Vulnerable Parameter:** `storeId` (and `productId`) sent to the application in **XML format** inside the request body.
* **Injection Type:** **UNION-based SQL injection** — query results are returned in the application's response (no blind/time/OOB techniques required).
* **Defense Mechanism:** A **WAF** flags and blocks requests containing obvious SQLi keywords (e.g., `UNION SELECT`), returning an "Attack detected" message.
* **Bypass Technique:** **XML entity encoding** (`hex_entities` / `dec_entities`) applied to the payload, typically with the **Hackvertor** Burp extension.

### Root Cause & Impact

* **Root Cause #1:** Insecure concatenation of user-controlled XML values into backend SQL queries without parameterized input filtering.
* **Root Cause #2:** Weak WAF implementation — it only matches literal keyword signatures in the raw request and fails to account for XML entity decoding done by the application layer.
* **End Goals:**
  * Bypass the WAF and use a UNION attack to dump the `users` table (`username` + `password`).
  * Retrieve the `administrator` account's credentials and log in to complete the lab.

---

## 3. Attack Methods & Techniques

### Method Details

* **Phase 1 (Identify the Injection Point):** Probe the `storeId` with a mathematical expression (`1+1`) and observe that it is **evaluated by SQL** — the response returns stock for store 2.
* **Phase 2 (Confirm the WAF):** Append `UNION SELECT NULL` and observe the request is **blocked** with "Attack detected".
* **Phase 3 (Determine Column Count):** Encode the UNION probe with XML entities and iterate column counts — you'll find the query returns **a single column** (more than one column → `0 units`).
* **Phase 4 (Bypass the WAF):** Use **Hackvertor → Encode → hex_entities** to obfuscate the payload as XML entity references.
* **Phase 5 (Exfiltrate Credentials):** Since only one column is available, concatenate username and password: `username || '~' || password` — the `~` acts as a separator.
* **Phase 6 (Log In):** Use the extracted `administrator` credentials on `/login` to solve the lab.

### Server Behavior

* **Blocked payload (raw SQLi):** HTTP response says **"Attack detected"** — the WAF drops the request.
* **Encoded payload (XML entities):** Normal HTTP response with the **injected UNION results rendered** (usernames/passwords visible, separated by `~`).
* **Too many columns in UNION:** The application returns `0 units`, signalling a column-count mismatch.

---

## 4. Step-by-Step Walkthrough

### Step 1: Capture the Stock Check Request

1. Open any product page in the shop and click **"Check stock"**.
2. Intercept the request in **Burp Proxy** and send it to **Repeater**.
3. Observe that the request is a `POST /product/stock` with an **XML body**:

```http
POST /product/stock HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Content-Type: application/xml
Cookie: session=[SESSION-COOKIE]

<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
    <productId>1</productId>
    <storeId>1</storeId>
</stockCheck>
```

### Step 2: Confirm SQL Evaluation in `storeId`

1. Replace the `storeId` value with a mathematical expression:

```xml
<storeId>1+1</storeId>
```

2. **Result:** The response returns the stock quantity for **store 2** — the value is being evaluated by the SQL engine, confirming the injection point (the app builds something like `SELECT stock FROM products WHERE productId=1 AND storeId=1+1`).

### Step 3: Trigger the WAF

1. Append a UNION probe:

```xml
<storeId>1 UNION SELECT NULL</storeId>
```

2. **Result:** The request is **blocked** — the response contains **"Attack detected"**. The WAF matches the literal `UNION SELECT` keyword in the raw body.

### Step 4: Install Hackvertor & Bypass the WAF

1. Go to **Extender → BApp Store**, search for **Hackvertor**, and click **Install** (it does not ship with Burp by default).
2. Back in Repeater, highlight the payload inside `storeId`:

```xml
<storeId>1 UNION SELECT NULL</storeId>
```

3. Right-click the highlighted text → **Extensions → Hackvertor → Encode → hex_entities**.
4. The payload is rewritten as XML entity references wrapped in Hackvertor tags, e.g.:

```xml
<storeId><@hex_entities>1 UNION SELECT NULL</@hex_entities></storeId>
```

> This decodes to something like `1 &#x55;NION &#x53;ELECT &#x4e;ULL` — no literal `UNION`/`SELECT` bytes remain in the raw request, so the WAF lets it through. The XML parser decodes the entities back to `1 UNION SELECT NULL` before the query executes.

5. Send the request.
6. **Result:** No more "Attack detected" — you receive a **normal application response**, confirming the WAF bypass works.

### Step 5: Determine the Column Count

1. Send the encoded `1 UNION SELECT NULL` and note the result — it returns stock data, meaning **1 column** works.
2. Try returning 2 columns (`1 UNION SELECT NULL, NULL`):
3. **Result:** The application returns **`0 units`** — an error state. The original query returns **exactly one column**.

### Step 6: Exfiltrate the Administrator's Credentials

1. Because only one column is available, use `||` to concatenate `username` and `password` with a `~` separator, and dump the whole `users` table:

```xml
<storeId><@hex_entities>1 UNION SELECT username || '~' || password FROM users</@hex_entities></storeId>
```

2. Send the request.
3. **Result:** The response now displays all registered users with their passwords, separated by `~`, e.g.:

```
administrator\~q1uuvho3ov5renjtp936
wiener\~peter
carlos\~montoya
```

4. **Extracted credentials:**
* **Username:** `administrator`
* **Password:** `<the string after the ~ for the administrator row>`

> ⚠️ **Note:** The password is randomly generated per lab instance — read yours from your own response.

### Step 7: Log in as Administrator

1. Navigate to `/login` on the application.
2. Log in with:
* **Username:** `administrator`
* **Password:** `<extracted password>`

3. The lab displays **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **Hackvertor must be installed first:** It's a **BApp Store extension** — it does NOT come pre-installed with Burp. If you don't see "Extensions → Hackvertor" in the right-click menu, install it from **Extender → BApp Store**.
* **This lab needs no Burp Pro:** Unlike Labs #15/#16 (Collaborator), the results are returned directly in the HTTP response, so it's fully solvable with **Burp Suite Community Edition**.
* **The WAF inspects raw bytes, not decoded XML:** That's the entire point — encode the payload as XML entities and the parser decodes it *after* the WAF check. You can encode the whole payload (hex_entities) or just the blocked keywords (e.g., replace `S` with `&#x53;` in `SELECT`).
* **Single column → concatenate:** `UNION SELECT NULL, NULL` returns `0 units`, which tells you the query returns one column. Don't fight it — use `|| '~' ||` to merge username and password into one column.
* **Pick a separator that won't appear in passwords:** `~` is the standard choice; any delimiter that isn't alphanumeric works for parsing the response.
* **Math evaluation is your confirmation:** `<storeId>1+1</storeId>` returning store 2's stock is the fastest way to prove SQL evaluation without tripping the WAF — do this before any UNION probing.
* **The `--` comment is optional here:** The official solution payload (`1 UNION SELECT username || '~' || password FROM users`) doesn't need it because the UNION result replaces the original result set; if you hit trailing-SQL syntax errors, append `--` after the payload and re-encode.
* **Hackvertor tags are client-side markers:** `<@hex_entities>...</@hex_entities>` is Hackvertor's instruction syntax. When you highlight and encode, Burp processes the tag and inserts the encoded text — send the request and only the entity-encoded payload should remain.
* **Don't over-encode:** If the response breaks or returns an error after encoding, verify the request still contains valid XML — every `&` must be a proper entity reference (`&#xHH;`), and the `<stockCheck>` structure must stay intact.