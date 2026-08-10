# Broken Access Control - Lab #2: Unprotected Admin Functionality with Unpredictable URL

**YouTube Tutorial:** [Broken Access Control - Lab #2 Unprotected admin functionality with unpredictable URL (Short Version)](https://youtu.be/7Jve11VySNU?si=tjcAz9TaObD3DA3R)

---

## 1. What is Unprotected Admin Functionality with Unpredictable URL?

### Core Concept

This lab is the natural follow-up to Lab #1. The flaw is identical — <cite index="61-1">this lab has an unprotected admin panel</cite> with no real authentication or authorization check — but the developers this time made the URL genuinely hard to guess, rather than something predictable like `/administrator-panel`. However, "hard to guess" is not the same as "properly protected." <cite index="61-1">It's located at an unpredictable location, but the location is disclosed somewhere in the application</cite> — meaning the client-side code itself leaks the very secret it was relying on.

```
            LAB #1 (predictable path)                  LAB #2 (unpredictable path)
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  /administrator-panel        │          │  /admin-a2f9x7 (random)     │
   │  found via robots.txt        │          │  robots.txt has NO hint     │
   └────────────────────────────┘          └─────────────┬──────────────┘
                                                          ▼
                                          ┌────────────────────────────┐
                                          │  View page source (Ctrl+U)  │
                                          │  Client-side JS reveals the │
                                          │  admin panel's real URL     │
                                          └─────────────┬──────────────┘
                                                          ▼
                                          ┌────────────────────────────┐
                                          │  Browse directly to the     │
                                          │  leaked URL — no auth check │
                                          │  = Broken Access Control ✅ │
                                          └────────────────────────────┘
```

**The page's own client-side JavaScript is the oracle** this time — obscurity fails not because the URL was easy to guess, but because the application handed the "secret" straight to the browser.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** <cite index="61-1">This lab has an unprotected admin panel.</cite>
* **Vulnerable Endpoint:** An unpredictable, randomized admin path (unlike Lab #1's static `/administrator-panel`).
* **Broken Access Control Mechanism:** Same as Lab #1 — no server-side authentication or role check is performed on the admin panel route. The only "protection" is that the URL is hard to guess.
* **Detection Channel:** <cite index="61-1">Reviewing the lab home page's source using Burp Suite or a web browser's developer tools reveals JavaScript that discloses the URL of the admin panel.</cite>
* **End Goal:** <cite index="61-1">Solve the lab by accessing the admin panel, and using it to delete the user carlos.</cite>

### Root Cause & Impact

* **Root Cause:** As in Lab #1, the developers again relied purely on obscurity — this time an unpredictable path instead of an easily-guessable one — but shipped the actual admin URL inside client-side JavaScript, defeating the obscurity the moment anyone views the page source.
* **Impact:** This demonstrates that "unpredictable" URLs are not a fix for missing access control — if the location has to be reachable by legitimate admins via some link or script, it will end up disclosed somewhere the client can read it, and any unauthenticated visitor gets the same access an admin would.

---

## 3. Attack Methods & Techniques (4-Phase Analysis)

* **Phase 1 — Reconnaissance of the Application:** Browse the site normally, and rule out `robots.txt` this time, since it discloses nothing useful for this variant.
* **Phase 2 — Inspect the Page Source for Disclosed URLs:** View the homepage's HTML/JavaScript source directly, looking for any hardcoded reference to an admin path.
* **Phase 3 — Access the Disclosed Admin Panel:** Browse directly to the leaked, unpredictable URL and confirm it loads without any authentication check.
* **Phase 4 — Exploit the Admin Functionality:** Use the panel to delete the target user and confirm the lab is solved.

### Server Behavior

* **`robots.txt` request:** Unlike Lab #1, this returns no useful hint for this variant — the disclosure mechanism has moved elsewhere.
* **Homepage source:** Contains embedded JavaScript referencing the actual (randomized) admin panel path.
* **Direct request to the leaked path:** Returns the full admin panel with no login redirect and no access-denied response — full, unauthenticated access granted immediately, exactly like Lab #1.

---

## 4. Step-by-Step Walkthrough

### Step 1 — Phase 1: Reconnaissance of the Application

1. Open the lab and browse normally — note that, just like Lab #1, there's no visible admin link in the navigation.
2. Try `/robots.txt` first out of habit (since it worked in Lab #1):

```http
GET /robots.txt HTTP/1.1
Host: [LAB-ID].web-security-academy.net
```

3. **Observation:** This time, `robots.txt` gives no useful path — the developers didn't make the same mistake twice. This confirms the disclosure vector must be somewhere else.

### Step 2 — Phase 2: Inspect the Page Source for Disclosed URLs

1. On the homepage, view the page source (Ctrl+U) or open browser DevTools.
2. Search the HTML and any linked/inline JavaScript for references to "admin".
3. **Result:** <cite index="61-1">The page source contains some JavaScript that discloses the URL of the admin panel</cite> — typically an unpredictable, randomized path such as `/admin-<random-string>`.

### Step 3 — Phase 3: Access the Disclosed Admin Panel

1. Take the leaked path from Step 2 and browse to it directly:

```http
GET /admin-<random-string> HTTP/1.1
Host: [LAB-ID].web-security-academy.net
```

2. **Result:** The admin panel loads immediately, with no login prompt and no access-denied response — confirming the same underlying flaw as Lab #1: the endpoint performs no real access control, it was only ever hidden by an unpredictable-but-disclosed URL.

### Step 4 — Phase 4: Exploit the Admin Functionality

1. On the admin panel, locate the list of users — the same `wiener` and `carlos` accounts as Lab #1.
2. Click the **Delete** button next to the user `carlos`.
3. **Result:** The user `carlos` is deleted successfully.
4. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **Don't stop at `robots.txt`:** This lab is specifically designed to teach that lesson — when one recon channel comes up empty, move to the next (page source, JS files, HTTP responses, directory brute-forcing) rather than assuming the target is actually secure.
* **Always check client-side JavaScript for hardcoded paths:** Developers often reference admin/internal routes in JS for convenience (e.g. conditionally rendering a link for admins), which leaks the path to every visitor's browser regardless of whether the link is actually shown.
* **"Unpredictable" is not "unauthenticated-safe":** A random string in a URL only raises the cost of *guessing* it — it does nothing to stop someone who finds it through legitimate application behavior, like this lab's own client-side code.
* **No Burp Suite strictly required:** Like Lab #1, browser DevTools alone (Ctrl+U or the Sources/Elements panel) are enough to find the leaked path and solve this lab — Burp's HTTP history is convenient but not mandatory here.
* **The vulnerable backend + frontend pattern:**

```javascript
// ❌ VULNERABLE — admin path leaked to every client via JS, and the
// route itself still has no server-side access control
if (isAdminLinkShown) {
  document.getElementById('nav').innerHTML += `<a href="/admin-a2f9x7">Admin</a>`;
}
```

```php
// ❌ VULNERABLE — no access control check on the route at all
Route::get('/admin-a2f9x7', 'AdminController@index');

// ✅ SAFE — enforce authentication AND authorization on every sensitive route,
// regardless of how unpredictable or hidden its URL is
Route::get('/admin-a2f9x7', 'AdminController@index')
    ->middleware(['auth', 'role:admin']);
```

* **Prevention summary:**
  * Never treat an unpredictable URL as a substitute for real, server-side authentication and authorization checks.
  * Never embed sensitive paths in client-side code, including JavaScript, HTML comments, or exposed source maps — anything shipped to the browser is visible to any visitor.
  * Apply access control centrally (middleware, security filter chains) so every sensitive route is protected by default, not opt-in per page.
  * Regularly audit both server-side routes and client-side bundles for accidental disclosure of internal or administrative paths.