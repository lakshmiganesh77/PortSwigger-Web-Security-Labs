# Broken Access Control - Lab #9: User ID Controlled by Request Parameter with Data Leakage in Redirect

**YouTube Tutorial:** [Broken Access Control - Lab #9 User ID controlled by request parameter with data leakage in redirect](https://youtu.be/aafn8BmHcIE?si=XfrztZEv_uL3UAz5)

---

## 1. What is Data Leakage in Redirect?

### Core Concept

At first glance, this lab looks like the access control check finally works: <cite index="114-1">if you change the id parameter to "carlos" and follow the request in a normal browser, you're redirected to the login page</cite> — access appears denied, just like it should be. But <cite index="115-1">this lab contains an access control vulnerability where sensitive information is leaked in the body of a redirect response.</cite> The server *does* build the full, unauthorized page — including Carlos's private data — and only *then* decides to redirect. It sends both the redirect status/header **and** the already-rendered page body in the same HTTP response. A browser silently follows the redirect and never shows you that body, but any tool that inspects the raw response (like Burp) sees everything that was "supposed" to be blocked.

```
            WHAT THE BROWSER SHOWS YOU                 WHAT ACTUALLY CAME BACK
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  GET /my-account?id=carlos  │          │  HTTP/1.1 302 Found          │
   │  → Browser auto-follows      │          │  Location: /login             │
   │  the redirect                │          │                                │
   │  → You land on the login      │          │  <html>                       │
   │  page, access looks denied   │          │    ...carlos's account page,  │
   └────────────────────────────┘          │    including his API key...   │
                                             │  </html>                      │
                                             │  ← the FULL leaked body is    │
                                             │  still sitting right here!    │
                                             └─────────────┬──────────────┘
                                                          ▼
                                          ┌────────────────────────────┐
                                          │  Inspect the raw response    │
                                          │  in Burp — Carlos's API key   │
                                          │  is sitting in the body       │
                                          │  = Broken Access Control ✅  │
                                          └────────────────────────────┘
```

**The redirect response's own body is the oracle** — the access-denied *decision* happened, but too late: the sensitive content had already been generated and attached before the redirect was sent.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** <cite index="115-1">This lab contains an access control vulnerability where sensitive information is leaked in the body of a redirect response.</cite>
* **Vulnerable Endpoint:** The same `/my-account?id=...` page family as Labs #7 and #8.
* **Broken Access Control Mechanism:** The server renders the target account's full page content *before* enforcing the access decision, then sends a `302` redirect to the login page — but never discards the already-generated (and already-attached) response body containing the unauthorized data.
* **Detection Channel:** Inspecting the **raw HTTP response** (not the browser's rendered view, which silently follows the redirect) reveals the full leaked page underneath the `302`.
* **Provided Credentials:** `wiener:peter`.
* **End Goal:** <cite index="115-1">Obtain the API key for the user carlos and submit it as the solution.</cite>

### Root Cause & Impact

* **Root Cause:** The application's rendering and its authorization check are sequenced incorrectly — the page is fully built (populating it with the target user's private data) before the access control decision is applied, and the resulting redirect fails to strip the now-irrelevant, sensitive body content.
* **Impact:** This demonstrates that a redirect alone is not proof that access was actually denied. Any tool or client that reads the full response — not just one that follows the `Location` header like a browser does — can retrieve data the application believed it had successfully blocked.

---

## 3. Attack Methods & Techniques (4-Phase Analysis)

* **Phase 1 — Log In and Note the id Parameter:** Log in with the provided credentials and observe your own account URL's `id` value.
* **Phase 2 — Attempt the Swap in a Normal Browser:** Change `id` to `carlos` directly in the browser and observe the apparent access-denied redirect to the login page.
* **Phase 3 — Re-Send the Request via Burp and Inspect the Raw Response:** Capture the same request in Burp Repeater and examine the full response body beneath the `302`, rather than trusting the browser's auto-followed result.
* **Phase 4 — Extract and Submit the Leaked API Key:** Locate Carlos's API key inside the leaked response body and submit it as the solution.

### Server Behavior

* **Browser-rendered result:** Appears to correctly deny access — the browser lands on the login page, showing nothing unusual.
* **Raw HTTP response (via Burp):** A `302 Found` status with a `Location: /login` header, but the **response body still contains the fully rendered `/my-account` page for `carlos`**, including his API key — the leak is entirely in the body the browser never displays.

---

## 4. Step-by-Step Walkthrough

### Step 1 — Phase 1: Log In and Note the id Parameter

1. Log in using the provided credentials `wiener:peter`.
2. Note your account URL:

```http
GET /my-account?id=wiener HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[WIENER-SESSION]
```

### Step 2 — Phase 2: Attempt the Swap in a Normal Browser

1. Edit the URL's `id` parameter directly in the browser's address bar to `carlos` and navigate to it.
2. **Observation:** <cite index="114-1">You are redirected to the login page,</cite> which looks like a correctly enforced access denial — this is the trap the lab is designed to illustrate.

### Step 3 — Phase 3: Re-Send the Request via Burp and Inspect the Raw Response

1. With Burp Proxy intercepting, repeat the same request (or send an earlier captured one to **Repeater**):

```http
GET /my-account?id=carlos HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[WIENER-SESSION]
```

2. Send it and look at the **raw response**, not the rendered/browser view.
3. **Result:** <cite index="114-1">Despite the response indicating a redirect (status code 302), the response body contains Carlos's "My Account" page, including his API key.</cite>

```http
HTTP/1.1 302 Found
Location: /login

<html>
  ...
  <input type="text" value="carlos" ...>
  <input type="text" value="[CARLOS-API-KEY]" ...>
  ...
</html>
```

### Step 4 — Phase 4: Extract and Submit the Leaked API Key

1. <cite index="110-1">Copy the API key from the leaked response body.</cite>
2. Go to the lab's **"Submit solution"** option in the Academy header and paste the API key as the answer.
3. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **Never trust what the browser shows you as proof of a security control working:** This lab exists specifically to teach that a redirect is not evidence that access was denied — the browser hides response bodies on redirects by default, which can make a broken control look like it's functioning correctly.
* **Always inspect raw HTTP responses when testing access control, not just the rendered page:** Burp's Repeater (or a tool like `curl -v` that doesn't auto-follow redirects) is essential here — this class of bug is invisible if you only ever interact with the app through a normal browser tab.
* **Look at ordering, not just outcomes, when reviewing access control implementations:** The underlying lesson generalizes — any code path that builds a sensitive response *before* checking authorization, then relies on a later step (like a redirect) to prevent disclosure, is fragile. The check needs to happen before any sensitive data is generated or attached to the response at all.
* **Compare with Labs #7 and #8:** Same vulnerable `id` parameter pattern and same end goal (Carlos's API key), but this lab's fix must happen earlier in the request lifecycle — check-then-render, not render-then-redirect.
* **The vulnerable backend pattern:**

```javascript
// ❌ VULNERABLE — renders the full page first, decides authorization after,
// and still attaches the rendered (sensitive) body to the redirect response
app.get('/my-account', (req, res) => {
  const targetUser = db.getUserByUsername(req.query.id);
  const body = renderAccountPage(targetUser); // sensitive data generated here
  if (targetUser.username !== req.session.username) {
    return res.redirect(302, '/login', body); // body still sent along with the redirect!
  }
  res.send(body);
});

// ✅ SAFE — check authorization BEFORE generating or attaching any sensitive content
app.get('/my-account', (req, res) => {
  const targetUser = db.getUserByUsername(req.query.id);
  if (targetUser.username !== req.session.username) {
    return res.redirect(302, '/login'); // no body, nothing leaked
  }
  const body = renderAccountPage(targetUser);
  res.send(body);
});
```

* **Prevention summary:**
  * Perform authorization checks before any sensitive data is fetched or rendered — never generate the protected content first and discard it later.
  * Ensure redirect responses never carry a response body containing data the redirect was meant to protect.
  * Test access control using raw HTTP inspection tools, not just a browser, since browsers hide exactly the kind of leak this lab demonstrates.
  * Apply the same "check first, render second" discipline across every code path, including error and redirect handling, not just the "happy path" response.