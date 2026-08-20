# File Upload Vulnerabilities - Lab #4: Web Shell Upload via Extension Blacklist Bypass

**YouTube Tutorial:** [File Upload Vulnerabilities - Lab #4 Web shell upload via extension blacklist bypass](https://youtu.be/c_NT4lIaNuA?si=Z1jFFsrw2VlFPs3A)

---

## 1. What is an Extension Blacklist Bypass?

### Core Concept

This lab adds a genuinely reasonable-sounding defense: the server maintains a list of dangerous file extensions — `.php`, and presumably common variants like it — and rejects any upload whose filename ends in one of them. Uploading `shell.php` directly gets flatly refused this time, with the app telling you plainly that PHP files aren't allowed.

The flaw isn't in the *idea* of a blacklist — it's that blacklists are fundamentally a "list everything bad" approach, and it's almost impossible to actually list everything bad. Rather than hunting for some obscure alternate PHP extension the blacklist forgot about, this lab's intended solution goes a level deeper: instead of trying to sneak a `.php` file past the filter, you upload a completely different kind of file — an Apache configuration file called `.htaccess` — which itself isn't blocked by the blacklist at all, because it doesn't look dangerous on its own. But that configuration file's *job* is to tell the web server "treat any file with this other extension as executable PHP." Once that `.htaccess` file is in place, you can upload something with a totally harmless-looking extension, and the server will now happily execute it as PHP anyway — because you told it to, using a file the blacklist never thought to block.

```
┌─────────────────────────────┐        ┌─────────────────────────────┐
│   DIRECT ATTEMPT (BLOCKED)   │        │   THE .htaccess TRICK        │
├─────────────────────────────┤        ├─────────────────────────────┤
│ Upload shell.php             │        │ Step 1: upload a file named  │
│                               │        │ .htaccess containing:        │
│ Blacklist sees ".php",       │        │ AddType application/         │
│ REJECTS the upload           │        │   x-httpd-php .l33t          │
└─────────────────────────────┘        └───────────────┬─────────────┘
                                                         │
                                                         ▼
                                        ┌─────────────────────────────┐
                                        │ .htaccess itself ISN'T on    │
                                        │ the blacklist — it's not     │
                                        │ ".php", so it uploads fine   │
                                        └───────────────┬─────────────┘
                                                         │
                                                         ▼
                                        ┌─────────────────────────────┐
                                        │ Step 2: upload shell.l33t    │
                                        │ (harmless-looking extension, │
                                        │ also not on the blacklist)   │
                                        └───────────────┬─────────────┘
                                                         │
                                                         ▼
                                        ┌─────────────────────────────┐
                                        │ Apache now treats ANY .l33t  │
                                        │ file as PHP, because YOU     │
                                        │ told it to via .htaccess     │
                                        │ = Remote Code Execution ✅   │
                                        └─────────────────────────────┘
```

The oracle here is a mismatch between what the blacklist is checking (a fixed list of "known dangerous" extensions) and what actually determines execution behavior on an Apache server (whatever the *active configuration* says, which you can rewrite yourself if `.htaccess` uploads are permitted in a directory Apache treats as configurable).

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

This lab contains a vulnerable image upload function. Certain file extensions are blacklisted, but this defense can be bypassed due to a fundamental flaw in the configuration of this blacklist. To solve the lab, upload a basic PHP web shell, then use it to exfiltrate the contents of the file `/home/carlos/secret`.

* **Provided Credentials:** `wiener:peter`
* **Hint given by the lab itself:** You need to upload two different files to solve this lab — this is a strong signal that the intended solution isn't a single clever filename, but a two-stage process.
* **What Changed From Earlier Labs:** Uploading `.php` directly is now flatly rejected with a clear "php files are not allowed" style message — this is a real, functioning check, not something bypassable via Content-Type or path tricks alone.
* **End Goal:** Same as before — get PHP code executing on the server, use it to read `/home/carlos/secret`, and submit the contents.

### Root Cause & Impact

The root cause is that a blacklist can only ever block extensions its authors thought to include — and it's fundamentally incomplete by design, since there's no way to enumerate every possible extension a web server might be configured to execute. Worse, in this specific lab, the blacklist doesn't account for the fact that Apache's behavior can itself be reconfigured *by an uploaded file* — meaning the attacker doesn't need to find a forgotten dangerous extension at all; they can simply create one, by uploading configuration that defines a brand-new "dangerous" extension of their own choosing, using a filename (`.htaccess`) the blacklist never considered a threat in the first place.

---

## 3. Attack Methods & Techniques (What I Tried, and What Actually Worked)

**First move — the obvious one.** Uploading `shell.php` exactly as in the earlier labs. This time it's rejected outright, with an explicit error saying PHP files aren't allowed. Checking the raw request and response in Burp confirms the rejection is happening server-side, based on the filename's extension specifically.

**Trying a couple of quick, obvious variants.** Testing whether the blacklist is exhaustive by trying a few well-known alternate PHP-executing extensions like `.php5` or `.phtml` (techniques that solved similar-looking labs elsewhere). Depending on how thorough this particular blacklist is, these may or may not be blocked too — but the lab's own hint (needing to upload *two* files) points toward a different, more deliberate technique rather than extension guessing.

**Checking what web server is actually running.** Looking at the response headers from any request reveals a `Server: Apache/...` header. Knowing this is Apache is the key piece of context — Apache supports per-directory configuration files named `.htaccess`, which, if the web server is configured to allow them (a setting called `AllowOverride`), let anyone who can place a file in that directory change how Apache behaves for that directory, without needing access to the main server configuration at all.

**The actual technique — reconfigure the server via `.htaccess`.** Since `.htaccess` isn't a PHP file and isn't on the blacklist, it should upload cleanly. Inside it, a single Apache directive can map any extension of your choosing to PHP execution:

```
AddType application/x-httpd-php .l33t
```

**Uploading the `.htaccess` file itself.** This requires a small trick during upload: the filename needs to be set to exactly `.htaccess` (no name before the dot — it's a dotfile), and since it's not really an image, the `Content-Type` needs to be something plausible like `text/plain` rather than an image type, since we're not pretending this one is an image at all, just getting it stored in the right directory.

**Uploading the actual shell, using the new extension.** With `.htaccess` in place declaring `.l33t` as executable PHP, uploading a file named `shell.l33t` containing the normal PHP payload — an extension that was never on the blacklist to begin with, because nobody would think to blacklist something so arbitrary.

**Confirming it worked.** Visiting the URL of the uploaded `.l33t` file directly executes it as PHP, exactly as if it had been a `.php` file all along — the blacklist never stood a chance, because the file that actually mattered for triggering the block (`shell.l33t`) was never dangerous-looking, and the file that reconfigured the server (`.htaccess`) was never PHP at all.

### Server Behavior

* **Uploading `shell.php` directly:** Rejected — "php files are not allowed" or similar, confirming a real, working extension blacklist.
* **Uploading a file named `.htaccess` containing `AddType application/x-httpd-php .l33t`:** Accepted — since `.htaccess` isn't a PHP file and isn't covered by the blacklist at all.
* **Uploading a file named `shell.l33t` (PHP code, arbitrary extension):** Accepted — `.l33t` was never on the blacklist either, since it's not a real, well-known executable extension on its own.
* **Requesting `shell.l33t`'s URL directly, after the `.htaccess` file is in place in the same directory:** Executes as PHP, since Apache now treats `.l33t` files in that directory as PHP scripts, per the directive you uploaded.

---

## 4. Step-by-Step Walkthrough

### Step 1: Confirm the Blacklist Blocks Direct PHP Uploads

1. Log in with `wiener:peter` and go to **"My account."**
2. Create a file named `shell.php` containing:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

3. Attempt to upload it via the avatar field.
4. **Observation:** The upload is rejected with a message indicating PHP files aren't allowed — confirming a genuine extension-based filter is active.

### Step 2: Identify the Web Server Software

1. In Burp's Proxy history, inspect the response headers of any request to the lab.
2. Look for a `Server` header, e.g. `Server: Apache/2.4.41 (Ubuntu)`.
3. This confirms Apache is in use, which is important because Apache specifically supports `.htaccess` files for per-directory configuration overrides — a feature not all web servers share.

### Step 3: Prepare the .htaccess File

1. On your machine, create a plain text file with **exactly** the filename `.htaccess` (starting with a dot, no name before it, no other extension).
2. Set its contents to:

```
AddType application/x-httpd-php .l33t
```

3. This single line tells Apache: "treat any file ending in `.l33t`, in this directory, as if it were a PHP script."

### Step 4: Upload the .htaccess File

1. Turn on Burp Proxy interception, or send the upload request straight to Repeater once captured.
2. Attempt to upload the `.htaccess` file via the avatar field.
3. In the intercepted/Repeater request, make sure of two things:
   - The `filename` field in the multipart body is set to exactly `.htaccess`.
   - The `Content-Type` header for this file part is set to something plausible and non-suspicious, such as `text/plain` — you're not pretending this is an image.
4. Forward or send the request.
5. **Result:** The upload succeeds — `.htaccess` was never on the extension blacklist, since it's not `.php` (or any of the other typical dangerous extensions the filter was written to catch).

### Step 5: Prepare and Upload the Web Shell With the New Extension

1. Create your PHP web shell file, but save it with the extension you defined in Step 3's `.htaccess` directive:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Save this as `shell.l33t`.

2. Upload this file via the same avatar upload field.
3. **Result:** This upload also succeeds — `.l33t` is not a recognized dangerous extension, so the blacklist has no reason to block it.

### Step 6: Trigger Execution

1. Find the URL where your files were stored — typically something like `/files/avatars/shell.l33t` (in the same directory as the `.htaccess` file you uploaded in Step 4, which is essential — the `.htaccess` directive only applies to files in the same directory it's placed in).
2. Visit this URL directly in your browser, or send a `GET` request to it via Burp Repeater.
3. **Result:** Because `.htaccess` reconfigured this directory to treat `.l33t` files as PHP, the server executes your code and the response contains the contents of `/home/carlos/secret`.

### Step 7: Submit the Solution

1. Copy the secret string from the response.
2. Click the lab banner's **"Submit solution"** button, paste the secret, and submit.
3. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Different Ways to Exploit This (Alternative Methods & Payloads)

**Simple extension-guessing, worth trying first before reaching for `.htaccess`:**

```
shell.php5
shell.phtml
shell.php3
shell.pht
shell.phar
```

Some real-world blacklists genuinely are incomplete and forget one or more of these legacy PHP-executing extensions — always worth a handful of quick attempts before assuming a more involved bypass like `.htaccess` reconfiguration is necessary.

**Automating extension-guessing with Burp Intruder, instead of manually trying each one:**

Send the upload request to Intruder, mark the file extension portion of the `filename` field as the payload position, and load a wordlist of known alternate executable extensions (`.php5`, `.phtml`, `.pht`, `.phar`, `.shtml`, `.asa`, `.cer`, `.asax` for IIS/ASP targets, etc.) as a **Simple list** payload. Compare response lengths/status codes across the results to spot which ones were accepted.

**Case-sensitivity bypass, if the blacklist check itself is case-sensitive but the filesystem/server isn't:**

```
shell.PHP
shell.pHp
shell.PhP
```

If the filter's comparison logic checks specifically for lowercase `.php` and the underlying OS/web server treats extensions case-insensitively (or Apache is configured to execute PHP regardless of case), varying the capitalization can slip past a naively-written check.

**Trailing character / null byte tricks, on older or specifically vulnerable server configurations:**

```
shell.php.
shell.php%00.jpg
shell.php%20
```

A trailing dot, a trailing space, or (on legacy PHP versions vulnerable to null byte injection) an embedded `%00` can sometimes cause the filter to see one thing while the filesystem stores another — these are largely historical bypasses at this point but worth knowing as a category.

**Redefining a different, more innocuous-sounding extension than `.l33t`,** in case a specific extension name itself triggers some other, separate filter:

```
AddType application/x-httpd-php .config
AddType application/x-httpd-php .settings
```

Choosing an extension that sounds like ordinary application data (rather than something that looks deliberately suspicious like `.l33t`) is a more realistic, stealthier choice in an actual assessment, since defenders reviewing logs are less likely to flag it immediately.

**Using an `.htaccess` directive to disable the extension check indirectly, rather than adding a new mapped extension**, in server configurations that support it:

```
RemoveHandler .php
AddType application/x-httpd-php .php
```

Some misconfigurations can be worked around by directly re-asserting PHP handling for the very extension that's blacklisted at the application layer — since `.htaccess` operates beneath the application's own validation logic, this can sometimes restore execution for the "blocked" extension itself, if the blacklist truly is only an application-layer check with no corresponding server-level restriction.

---

## 6. If This Existed in the Real World

Extension blacklists are a widely recognized anti-pattern in real security engineering precisely because they can never be complete, and the `.htaccess` reconfiguration trick specifically has a long, well-documented history of real-world exploitation:

* **Countless real Apache-hosted CMS and forum platforms** running with `AllowOverride All` (permitting `.htaccess` files to override server configuration) alongside an upload feature that blacklists only `.php` and a couple of obvious variants have been compromised this exact way — it's common enough to appear in standard penetration testing methodologies and OWASP guidance as a named technique.
* **Shared hosting environments** are particularly susceptible, since many hosting providers enable `.htaccess` support broadly for legitimate customization purposes (custom redirects, caching rules), inadvertently giving any application with a file upload vulnerability on that hosting account a path to full reconfiguration of how that directory is served.
* **Security guidance from Apache itself, and virtually every web application security standard (OWASP included),** explicitly recommends disabling `.htaccess` override support (`AllowOverride None`) in production and instead managing all configuration centrally, specifically because of this exact class of abuse — the fact that this recommendation exists as a standard hardening step is itself evidence of how often this pattern has been exploited in real deployments.

---

## 7. Pro-Tips & Common Pitfalls

* **Pay attention to lab hints — "you need to upload two files" is a genuine clue, not filler text.** When a lab or a real target explicitly signals that a multi-stage approach is needed, resist the urge to keep hammering on single-file extension tricks; it's telling you the intended path requires reconfiguring something first.
* **The `.htaccess` file must land in the *same directory* as the file you want executed.** Apache's per-directory configuration only applies to the directory it's placed in (and subdirectories, by default) — uploading `.htaccess` to one location and your shell to a different one won't work.
* **Set a plausible Content-Type for the `.htaccess` upload, and don't overthink the filename.** It genuinely is just `.htaccess`, starting with the dot — this is a legitimate Unix/Apache convention for a hidden configuration file, not a typo or placeholder.
* **Identify the web server technology early in any file upload assessment.** The `.htaccess` technique is Apache-specific; different servers (IIS, Nginx) have their own equivalent (or entirely different) configuration mechanisms and dangerous-extension quirks, so knowing the target stack shapes which bypass techniques are even worth attempting.
* **The vulnerable backend pattern:**

```apache
# ❌ VULNERABLE — AllowOverride enabled broadly, letting any uploaded
# .htaccess file reconfigure how this directory serves and executes files
<Directory "/var/www/files">
    AllowOverride All
</Directory>
```

```php
// ❌ VULNERABLE — blocks a fixed list of "known dangerous" extensions,
// with no consideration for files that could reconfigure server behavior
$blacklist = ['php', 'php3', 'php4', 'php5', 'phtml'];
$ext = pathinfo($_FILES['avatar']['name'], PATHINFO_EXTENSION);
if (in_array(strtolower($ext), $blacklist)) {
    die('File type not allowed');
}
```

```apache
# ✅ SAFE — disable .htaccess override support entirely for upload directories,
# and explicitly deny script execution regardless of extension
<Directory "/var/www/files">
    AllowOverride None
    <FilesMatch "\.(php|php3|php4|php5|phtml|pht|phar)$">
        Require all denied
    </FilesMatch>
</Directory>
```

* **Prevention summary:**
  * Use an allow-list of explicitly permitted, safe file extensions rather than a blacklist of dangerous ones — allow-lists are complete by construction, blacklists never are.
  * Disable `AllowOverride`/`.htaccess` support for any directory that accepts user-uploaded files, so uploaded content can never reconfigure how that directory is served.
  * Enforce script-execution restrictions at the web server configuration level (as a `Require all denied` rule, or by disabling script handlers entirely for that directory), independent of and in addition to any application-layer filename validation.
  * Combine multiple, independent layers of defense — content inspection, safe storage location, disabled execution, and randomized filenames — since any single layer, including a blacklist, can eventually be found incomplete.