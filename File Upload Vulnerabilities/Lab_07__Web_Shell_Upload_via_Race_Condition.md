# File Upload Vulnerabilities - Lab #7: Web Shell Upload via Race Condition

**YouTube Tutorial:** [File Upload Vulnerabilities - Lab #7 Web shell upload via race condition](https://youtu.be/SSXuHCzXa1c?si=Bqtp2hBXl_LEhEBR)

---

## 1. What is a Race Condition in File Upload Handling?

### Core Concept

This final lab in the module has the strongest defense yet — genuinely strong. None of the earlier techniques work here: Content-Type spoofing, path traversal, extension blacklist tricks, null bytes, even the polyglot approach all fail. The server does something more thorough: it accepts the uploaded file, checks whether it's a genuine, safe image (likely scanning for malware and verifying real image structure), and if the check fails, it **deletes the file**.

The vulnerability isn't in *what* gets checked — it's in *when*. Look closely at the order of operations the server actually performs: the file is written to disk **first**, and only **afterward** does a separate validation step run and decide whether to delete it. Between those two steps, there's a brief window — sometimes just milliseconds — during which a genuinely malicious file sits on disk, fully uploaded, before the server gets around to removing it. If you can request that file's URL during that exact window, before the deletion happens, the server will execute it, because at that moment in time, the file is simply... there.

This is a classic **race condition**: the outcome depends on the relative timing of two independent operations (the deletion check vs. your own request) racing against each other. A human clicking a link by hand has zero chance of winning this race — the window is far too small. But by firing many requests at once, all timed to arrive as close to simultaneously as possible, you can reliably win that race at least once, which is all you need.

```
┌─────────────────────────────┐        ┌─────────────────────────────┐
│   NORMAL TIMELINE (deleted)  │        │   RACE CONDITION EXPLOIT     │
├─────────────────────────────┤        ├─────────────────────────────┤
│ t=0ms: Upload shell.php      │        │ t=0ms: Fire the SAME upload  │
│ t=0ms: File written to disk  │        │ request AND several GET      │
│ t=5ms: Validation runs,      │        │ requests to the file's URL,  │
│ file fails the check         │        │ all queued to send at        │
│ t=6ms: File is DELETED       │        │ nearly the exact same time   │
│                               │        │                               │
│ A normal, one-at-a-time      │        │                               │
│ visit to the URL always      │        │                               │
│ arrives AFTER deletion       │        │                               │
└─────────────────────────────┘        └───────────────┬─────────────┘
                                                         │
                                                         ▼
                                        ┌─────────────────────────────┐
                                        │ File genuinely exists on     │
                                        │ disk for a few milliseconds  │
                                        │ before validation deletes it │
                                        └───────────────┬─────────────┘
                                                         │
                                                         ▼
                                        ┌─────────────────────────────┐
                                        │ At least ONE of the raced    │
                                        │ GET requests lands inside    │
                                        │ that tiny window, BEFORE     │
                                        │ deletion — executes the      │
                                        │ shell = Remote Code           │
                                        │ Execution ✅                  │
                                        └─────────────────────────────┘
```

The oracle here isn't a check you can trick with cleverly-formed input at all — it's **timing itself**. The check is entirely correct and would catch your file every single time, if you only ever requested it one at a time, slowly. The vulnerability only exists because of the gap between two operations that were never made atomic (i.e., guaranteed to happen as a single, uninterruptible unit).

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

This lab's file upload functionality is vulnerable to a race condition, which enables you to bypass the website's file type checks. To solve the lab, upload a basic PHP web shell, then use it to exfiltrate the contents of the file `/home/carlos/secret`.

* **Provided Credentials:** `wiener:peter`
* **What Changed From Earlier Labs:** Every previous technique (Content-Type spoofing, path traversal, blacklist bypasses, obfuscated extensions, polyglot files) has been shut down — the server appears to genuinely, correctly detect and reject any malicious file, no matter how it's disguised.
* **The Bypass Tool:** Burp Repeater's built-in **"Send group in parallel"** feature (a single-packet attack, native to modern Burp Suite) — no extension required. Turbo Intruder is also covered further below as an alternative for anyone on an older Burp version without this feature.
* **End Goal:** Get PHP code to execute during the brief window before server-side validation deletes it, use that execution to read `/home/carlos/secret`, and submit the contents.

### Root Cause & Impact

The root cause is a classic **Time-of-Check to Time-of-Use (TOCTOU)** flaw: the file is written to disk, and only afterward is it checked and potentially removed. The two operations — "save the file" and "validate and possibly delete the file" — are not performed as a single, atomic unit, leaving a real, exploitable gap between them. This class of bug generalizes far beyond file uploads to any system where a resource is created before being validated, checked out before being confirmed available, or debited before a balance check completes.

---

## 3. Attack Methods & Techniques (What I Tried, and What Actually Worked)

**First move — the polyglot trick from Lab #6.** Given how thoroughly the earlier techniques have escalated, trying the ExifTool-based polyglot approach first seems reasonable. This time it fails outright — the upload either gets rejected immediately, or the file disappears by the time you try to access it. This tells you the server's validation is doing genuine malware/content scanning, not just a magic-byte check this time.

**Reading the lab's own hint carefully.** The lab description explicitly calls out a race condition as the vulnerability class here — a strong, direct signal about what to actually look for, rather than continuing to iterate on upload-content tricks.

**Understanding the actual check being performed.** Some versions of this lab even show you a PHP code snippet resembling the server's own logic — something like: the file is moved to its final destination first (`move_uploaded_file`), and only afterward does the code call something like `checkViruses()` and `checkFileType()`, deleting the file if either check fails. Seeing this (or inferring it from behavior) confirms the two-step, non-atomic nature of the validation.

**Confirming the file exists briefly.** A plain `shell.php` upload, followed immediately by a manual, single `GET` request to its expected URL, still fails — a human clicking "upload" and then clicking a link is always far too slow to beat the deletion. This doesn't disprove the race condition; it just confirms the window is too small for manual exploitation.

**The actual technique — race the upload against many simultaneous GET requests.** Rather than trying to win the race with a single well-timed request, the practical approach is to fire the upload request together with a batch of GET requests to the shell's expected URL, all sent as close to simultaneously as possible. Even if most of those GET requests arrive too late (after deletion), the odds are good that at least one lands inside the brief window while the file still exists — and that one request is all that's needed to get the PHP code to execute and leak the secret.

**Turbo Intruder is the right tool for this, not standard Intruder.** Standard Burp Intruder sends requests sequentially, with enough overhead between them that the timing precision needed here usually isn't achievable. Turbo Intruder is purpose-built for exactly this kind of tight-timing, high-concurrency request racing, using its own Python-based scripting engine to queue and release requests with much finer control.

### Server Behavior

* **Uploading `shell.php` via any of the earlier labs' bypass techniques:** Fails or is cleaned up — this server's validation appears to be robust against all previously effective tricks.
* **Uploading `shell.php` and then requesting its URL once, manually, after a normal human delay:** Fails — by the time a person can click through to the URL, the file has already been validated and deleted.
* **Firing the upload request together with several GET requests to the shell's URL, all released at nearly the same instant:** At least one of the GET requests, by chance, lands within the tiny window between file creation and deletion — that request receives the executed PHP output before the file is removed.

---

## 4. Step-by-Step Walkthrough

### Step 1: Confirm the Upload Location Pattern

1. Log in with `wiener:peter` and go to **"My account."**
2. Upload a genuine, valid image as your avatar first, just to observe normal behavior.
3. In Burp's **Proxy → HTTP history**, find the `GET` request your browser makes to fetch that image back, e.g.:

```
GET /files/avatars/<YOUR-IMAGE-FILENAME>
```

4. This confirms the exact URL pattern your malicious file will also be stored under, once uploaded.

### Step 2: Prepare the PHP Web Shell

1. Create a file named `exploit.php` (or any name) containing:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

### Step 3: Attempt the Upload Normally to Confirm It's Blocked

1. Attempt to upload `exploit.php` as your avatar through the normal form.
2. **Observation:** The upload appears to succeed at first, or the server otherwise behaves as if it accepted a valid file — but if you wait even a short moment and then request the file's URL, it's already gone or was never genuinely executable. This confirms the file is being deleted shortly after upload, not rejected outright at the point of submission.

### Step 4: Send the Upload Request to Repeater

1. With Burp Proxy intercepting, attempt the `exploit.php` upload once more.
2. Locate the `POST /my-account/avatar` request in your Proxy history.
3. Right-click this request and select **Send to Repeater**. This opens it as a new tab in Repeater — rename this tab something clear, like **UPLOAD**, by double-clicking its label.

### Step 5: Send the File-Fetch Request to Repeater

1. Find (or manually construct) a `GET` request to the expected URL of your uploaded shell, based on the pattern observed in Step 1, e.g.:

```
GET /files/avatars/exploit.php
```

2. Send this request to Repeater as well, opening it as a **separate** new tab. Rename this tab **FETCH**.
3. At this point you should have two distinct tabs open in Repeater: **UPLOAD** (the POST request) and **FETCH** (the GET request).

### Step 6: Group the Two Requests Together

1. In the Repeater tab bar, select both the **UPLOAD** and **FETCH** tabs together — hold **Ctrl** (or **Cmd** on macOS) and click each tab to multi-select them.
2. Right-click on the selected tabs and choose **Add tab to new group** (or **Group requests**, depending on your Burp version).
3. Give the new group a name, e.g. `race-condition-upload`.
4. Both requests now live inside a single Repeater group, which you can view together as a set.

### Step 7: Duplicate the FETCH Request Within the Group

1. Since a single upload request racing against only one fetch request has fairly low odds of winning, duplicate the **FETCH** tab several times within the same group — right-click the **FETCH** tab and choose **Duplicate tab**, repeating this four or five times.
2. You should now have your group containing: one **UPLOAD** tab, and five or six **FETCH** tabs (all identical GET requests to the same shell URL).
3. Having multiple simultaneous GET attempts significantly increases the odds that at least one of them lands inside the tiny race window before the file gets deleted.

### Step 8: Set the Group's Execution Mode to Parallel

1. With the group selected, look for the **send group in sequence / send group in parallel** toggle, usually found near the **Send** button at the top of the grouped Repeater view.
2. Change this setting to **Send group in parallel (single-packet attack)**.
3. This mode holds back the final bytes of every request in the group and releases them all at essentially the same network instant — the same underlying idea as Turbo Intruder's "gate" mechanism, but built directly into Repeater with no scripting or extension required.

### Step 9: Send the Group and Trigger the Race

1. Click the group's **Send group** button.
2. All requests in the group — the upload and every duplicated fetch — fire together, arriving at the server within milliseconds of each other.
3. If it doesn't succeed on the first attempt, simply click **Send group** again — race conditions are probabilistic, and it's completely normal to need a handful of attempts before one of the fetch requests wins the race.

### Step 10: Check Each Response in the Group

1. After sending, Repeater shows the response for every request in the group, typically in a tabbed or list view alongside each request.
2. Click through each of the **FETCH** responses one at a time.
3. Most will likely show a 404 or similar "not found" response — this just means that particular request lost the race and arrived after the file had already been deleted.
4. **At least one** FETCH response (across this attempt or a repeated one) should show a `200 OK` containing the actual executed PHP output — the contents of `/home/carlos/secret`.

### Step 11: Submit the Solution

1. Copy the secret string from the successful response.
2. Click the lab banner's **"Submit solution"** button, paste the secret, and submit.
3. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Different Ways to Exploit This (Alternative Methods & Payloads)

**Using standard Burp Intruder instead of Turbo Intruder, for those less comfortable with the Python scripting approach:**

While less precise, a simplified version of this attack can sometimes work using regular Burp Intruder:

1. Send the upload request to standard **Intruder**.
2. Under **Payloads**, set **Payload type** to **Null payloads**.
3. Under **Payload settings**, enable **Continue indefinitely**.
4. Simultaneously, in a separate Repeater tab (or a second browser), repeatedly send GET requests to the expected shell URL while the Intruder attack is running in the background.
5. This is far less reliable than Turbo Intruder's single-packet-style timing, but can occasionally succeed, especially against a target with a comparatively larger race window, and is a reasonable fallback for understanding the concept without installing an additional extension.

**Increasing `concurrentConnections` for a tighter race**, if the default value doesn't reliably win the race on a given attempt:

```python
engine = RequestEngine(
    endpoint=target.endpoint,
    concurrentConnections=15,
)
```

More concurrent connections and more queued GET requests in the `for` loop both increase the odds that at least one request lands inside the vulnerable window, at the cost of generating more overall traffic.

**Racing multiple different payload files simultaneously**, if you're unsure which of several obfuscation techniques will best survive the validation window:

Combine this lab's timing-based race with an earlier lab's obfuscation approach (e.g., a polyglot or double-extension file) as extra insurance — even a file that would eventually fail content validation only needs to survive long enough for one raced GET request to hit it.

**Using Turbo Intruder's built-in single-packet-attack template as a starting point**, rather than writing the script from scratch:

Turbo Intruder ships with several example script templates accessible from a dropdown in its editor, including ones specifically designed for race condition testing (similar in spirit to the `race-single-packet-attack.py` template used in other race-condition-focused labs) — starting from a maintained template and adapting it to this specific request pair is often faster and less error-prone than writing the full script manually.

**Using the Turbo Intruder extension instead of Repeater's built-in grouping**, for older Burp versions or finer-grained control over concurrency:

If your Burp Suite version predates the native "Send group in parallel" Repeater feature, or you want more precise control over concurrency and timing than the Repeater UI exposes, Turbo Intruder remains a fully viable alternative approach to the exact same race condition:

1. In Burp Suite, go to the **Extensions** tab, then **BApp Store**, search for **"Turbo Intruder"**, and click **Install**.
2. Capture the `POST /my-account/avatar` upload request, right-click it, and select **Extensions → Turbo Intruder → Send to Turbo Intruder**.
3. In the Turbo Intruder window's Python editor, replace the default script with:

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=10,
    )

    request1 = '''<YOUR-POST-REQUEST-HERE>'''
    request2 = '''<YOUR-GET-REQUEST-HERE>'''

    # the 'gate' argument blocks the final byte of each request
    # until openGate is invoked, letting us line them all up first
    engine.queue(request1, gate='race1')

    for x in range(5):
        engine.queue(request2, gate='race1')

    # wait until every 'race1'-tagged request is ready,
    # then release the final byte of ALL of them at once
    engine.openGate('race1')

    engine.complete(timeout=60)

def handleResponse(req, interesting):
    table.add(req)
```

4. Replace `<YOUR-POST-REQUEST-HERE>` with the full raw upload request, and `<YOUR-GET-REQUEST-HERE>` with the full raw `GET /files/avatars/exploit.php` request.
5. This script achieves conceptually the same thing as Repeater's grouped parallel send: `engine.queue(..., gate='race1')` holds each request's final byte back, and `engine.openGate('race1')` releases all of them — the upload plus five duplicate GET attempts — at essentially the same instant. Click **Attack** to run it, then check the results table for a `200 OK` response among the GET requests, exactly as you would check each response in a Repeater group.
6. Turbo Intruder's main advantage over Repeater's native grouping is finer control — adjustable `concurrentConnections`, easy scripted looping for many more simultaneous attempts than manually duplicating tabs, and reusable templates — useful if the race window on a given target turns out to be tighter than what a handful of grouped Repeater requests can reliably win.

---

## 6. If This Existed in the Real World

Race conditions in file upload validation, and TOCTOU bugs more broadly, are a well-established and actively exploited real-world vulnerability class:

* **Real-world "check-then-act" vulnerabilities** are extremely common across many domains beyond file uploads — payment processing systems that debit a balance before confirming sufficient funds, ticket/inventory systems that reserve an item before confirming stock, and authentication systems that grant a session token before fully completing verification have all had documented, exploitable race conditions with real financial or security impact.
* **PayPal, Starbucks, and numerous fintech platforms** have had publicly disclosed race condition vulnerabilities in their transaction and balance-checking logic, allowing attackers to submit multiple simultaneous requests (transfers, redemptions, purchases) that each individually pass a balance check performed *before* any of the concurrent requests had actually been applied — effectively spending the same balance multiple times.
* **File upload race conditions specifically** have shown up in real bug bounty reports against platforms performing server-side malware scanning or content validation after (rather than before or atomically with) making an uploaded file publicly accessible — precisely the pattern this lab teaches, and PortSwigger's own research (the "Smashing the state machine" and subsequent race condition research) significantly raised awareness of just how exploitable these timing windows can be, even ones previously assumed "too small to matter."

---

## 7. Pro-Tips & Common Pitfalls

* **If every prior technique in a module suddenly stops working, take that as a signal, not a dead end.** This lab is specifically designed to teach that lesson — when content-based, extension-based, and encoding-based bypasses all fail against a target that otherwise looks identical to earlier labs, it's worth considering that the vulnerability has moved to an entirely different category (timing) rather than assuming the target is simply unexploitable.
* **A manual, single GET request will almost never win this race — don't conclude the technique failed from that alone.** The whole point of grouping requests and sending them in parallel (whether via Repeater's native grouping or Turbo Intruder's gated queuing) is that human reaction time, and even standard sequential request sending, are both far too slow for the millisecond-scale windows involved.
* **Check every response in the group, not just the first one, and don't hesitate to resend the group if the first attempt fails.** Most of the raced GET requests will lose — that's expected and normal. Success looks like *one* request out of the group having a distinctly different, successful response; don't assume failure just because most of them show 404s, and don't assume the technique doesn't work just because the first attempt didn't land a hit.
* **This is the final lab in the File Upload Vulnerabilities module — it's meant to be the hardest, and rightly so.** Race conditions require an entirely different mental model than the previous six labs' input-crafting techniques; it's normal for this one to take longer to click, and understanding the general concept of TOCTOU bugs pays off well beyond file upload scenarios specifically.
* **The vulnerable backend pattern:**

```php
// ❌ VULNERABLE — the file is fully written to disk and immediately
// publicly accessible BEFORE validation runs; deletion only happens
// after the fact, leaving a genuine window of exploitability
$target_file = "avatars/" . $_FILES["avatar"]["name"];
move_uploaded_file($_FILES["avatar"]["tmp_name"], $target_file); // file is live NOW
if (checkViruses($target_file) && checkFileType($target_file)) {
    echo "The file " . $target_file . " has been uploaded.";
} else {
    unlink($target_file); // deleted only AFTER the fact — too late for a raced request
    echo "Invalid file.";
}

// ✅ SAFE — validate BEFORE the file is ever placed in a publicly
// accessible, executable location; use a temporary, non-servable
// staging directory for validation, then move only verified-safe files
$temp_file = "/private_staging/" . uniqid();
move_uploaded_file($_FILES["avatar"]["tmp_name"], $temp_file); // NOT web-accessible
if (checkViruses($temp_file) && checkFileType($temp_file)) {
    rename($temp_file, "avatars/" . generateRandomFilename() . ".jpg"); // only now made public
} else {
    unlink($temp_file);
}
```

* **Prevention summary:**
  * Never place an uploaded file in a publicly accessible, executable location until *after* it has been fully validated — validate first, in a private staging area, and only then move verified-safe files into their final, servable location.
  * Treat "save" and "validate-then-possibly-delete" as needing to be effectively atomic from an external observer's perspective — if a gap between them is unavoidable, ensure nothing outside the validation process can access the file during that gap.
  * Apply the same TOCTOU-awareness to any other check-then-act pattern in an application, not just file uploads — balance checks, inventory checks, and permission checks are all common candidates for the same underlying flaw.
  * Consider using atomic filesystem or database operations (transactions, file locks, or moving files only after validation completes) wherever a resource's availability and its validation status need to stay consistent with each other.