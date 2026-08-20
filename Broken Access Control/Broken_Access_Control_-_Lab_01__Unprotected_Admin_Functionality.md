# Broken Access Control - Lab #1: Unprotected Admin Functionality

**YouTube Tutorial:** [Broken Access Control - Lab #1 Unprotected admin functionality (Short Version)](https://youtu.be/qJ8mtm_G40E?si=7-qw4uA2MGEFgOQH)

---

## 1. What is Unprotected Admin Functionality?

### Core Concept

This is the simplest possible **Broken Access Control** flaw: sensitive, privileged functionality exists on the site, but the application applies **no access control check at all** to it — no login requirement, no role check, nothing. The only thing "protecting" it is that its URL isn't linked anywhere in the normal site navigation. This is **security through obscurity**, and it fails the moment an attacker finds the URL through any other means — in this lab, via `robots.txt`.

```
            NORMAL USER                                ATTACKER
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  Browses site normally      │          │  Checks /robots.txt          │
   │  Never sees an admin link   │          │  Disallow: /administrator-  │
   │                              │          │  panel                      │
   └─────────────┬──────────────┘          └─────────────┬──────────────┘
                 ▼                                       ▼
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  No way to reach the        │          │  Browses directly to        │
   │  admin panel through the UI │          │  /administrator-panel       │
   └────────────────────────────┘          └─────────────┬──────────────┘
                                                         ▼
                                          ┌────────────────────────────┐
                                          │  Admin panel loads —        │
                                          │  NO login or role check!    │
                                          │  = Broken Access Control ✅ │
                                          └────────────────────────────┘
```

**`robots.txt` is the oracle** — a file meant to tell search engines what *not* to index ends up telling an attacker exactly where to look.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** <cite index="36-1">This lab has an unprotected admin panel.</cite>
* **Vulnerable Endpoint:** <cite index="34-1">/administrator-panel</cite> — disclosed via <cite index="34-1">robots.txt, which disallows all bots from indexing /administrator-panel.</cite>
* **Broken Access Control Mechanism:** The admin panel performs no authentication or authorization check whatsoever. Any visitor — logged in or not — who knows or guesses the URL can load it directly.
* **Detection Channel:** The site's own `robots.txt` file, which is meant purely as a crawler directive but inadvertently reveals the hidden path.
* **End Goal:** <cite index="36-1">Solve the lab by deleting the user carlos.</cite>

### Root Cause & Impact

* **Root Cause:** The developers relied on the admin panel's URL simply not being linked anywhere as its only protection, rather than enforcing a real server-side access control check (e.g. verifying an authenticated session with an admin role).
* **Impact:** Anyone who discovers the URL — through `robots.txt`, page source, JavaScript files, directory brute-forcing, or even a search engine that ignored the disallow directive — gets full, unauthenticated access to administrative functions, including here the ability to delete user accounts.

---

## 3. Attack Methods & Techniques (4-Phase Analysis)

* **Phase 1 — Reconnaissance of the Application:** Browse the site normally first to confirm there's no visible link to any admin functionality in the UI.
* **Phase 2 — Check robots.txt for Disclosed Paths:** Request `/robots.txt` directly, since it's a standard, always-worth-checking file that often leaks paths the developer wanted hidden from search engines (and, unintentionally, from casual observation).
* **Phase 3 — Access the Disclosed Admin Panel:** Browse directly to the disclosed path and confirm it loads with no authentication prompt or access-denied response.
* **Phase 4 — Exploit the Admin Functionality:** Use the now-accessible panel to delete the target user and confirm the lab is solved.

### Server Behavior

* **Normal browsing:** No link to the admin panel appears anywhere in the site's navigation, HTML, or client-side JavaScript.
* **`robots.txt` request:** Returns a `Disallow` entry pointing at the exact hidden admin path — a direct, unintentional leak.
* **Direct request to the hidden path:** Returns the full admin panel with no login redirect, no 401/403 error — full access granted immediately.

---

## 4. Step-by-Step Walkthrough

### Step 1 — Phase 1: Reconnaissance of the Application

1. Open the lab and browse the shop normally — view products, the homepage, and any account pages.
2. Check the page source (Ctrl+U) for any hints of an admin link or comment.
3. **Observation:** No admin link is visible anywhere in the UI or source — <cite index="39-1">searching for a hint comment in posts, user details, the homepage, and the cart page turns up nothing.</cite> This confirms the panel is hidden, not linked.

### Step 2 — Phase 2: Check robots.txt for Disclosed Paths

1. Append `/robots.txt` to the lab's base URL and request it directly:

```http
GET /robots.txt HTTP/1.1
Host: [LAB-ID].web-security-academy.net
```

2. **Result:** <cite index="34-1">The robots.txt file discloses: User-agent: * Disallow: /administrator-panel</cite> — telling all bots not to index `/administrator-panel`, which is exactly the hidden path an attacker needs.

### Step 3 — Phase 3: Access the Disclosed Admin Panel

1. Browse directly to the disclosed path:

```http
GET /administrator-panel HTTP/1.1
Host: [LAB-ID].web-security-academy.net
```

2. **Result:** <cite index="38-1">Browsing to https://<random-string>.web-security-academy.net/administrator-panel loads the unprotected admin panel</cite> — no login prompt, no access-denied page, nothing standing in the way. This confirms the Broken Access Control vulnerability: privileged functionality reachable by anyone who knows the URL.

### Step 4 — Phase 4: Exploit the Admin Functionality

1. On the admin panel, locate the list of users — <cite index="32-1">two users are shown, wiener and carlos.</cite>
2. <cite index="32-1">Click the Delete button next to the user carlos.</cite>
3. **Result:** <cite index="32-1">The user carlos is deleted successfully.</cite>
4. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **Always check `robots.txt` early:** It's one of the first things worth requesting on any target — a file meant to guide search engine crawlers frequently doubles as a map of "sensitive" paths the developers wanted to keep quiet, without actually protecting them.
* **Don't rely only on `robots.txt`:** In follow-up labs in this series (e.g. "Unprotected admin functionality with unpredictable URL"), `robots.txt` reveals nothing, and the hidden path is instead disclosed elsewhere — <cite index="35-1">it's located at an unpredictable location, but the location is disclosed somewhere in the application,</cite> such as in page source, JavaScript files, or HTML comments. Always check multiple recon channels, not just one.
* **Obscurity is not access control:** A URL being unlinked or "hidden" is not a substitute for a real authentication/authorization check. If the endpoint doesn't verify who's asking, it's exposed to anyone who finds it by any means — search engines, brute-forcing, leaked documentation, or simple guessing.
* **No Burp Suite strictly required:** This particular lab can be solved with just a browser, since it only involves requesting known paths directly — though using Burp's **Proxy > HTTP history** to review requests as you browse is still good practice for spotting other leads.
* **The vulnerable backend pattern:**

```php
// ❌ VULNERABLE — no access control check on the route at all
Route::get('/administrator-panel', 'AdminController@index');

// ✅ SAFE — enforce authentication AND authorization on every sensitive route
Route::get('/administrator-panel', 'AdminController@index')
    ->middleware(['auth', 'role:admin']);
```

* **Prevention summary:**
  * Never rely on an unpredictable or unlinked URL as your only protection — enforce real, server-side authentication and role checks on every sensitive endpoint.
  * Apply access control centrally (e.g. middleware, a security filter chain) rather than per-page, so no route can be accidentally left unprotected.
  * Exclude sensitive paths from `robots.txt` entirely if possible, or accept that anything listed there should still be fully access-controlled regardless of whether it's disallowed for crawlers.
  * Regularly audit all routes/endpoints in the application to confirm every privileged action requires proper authorization.
