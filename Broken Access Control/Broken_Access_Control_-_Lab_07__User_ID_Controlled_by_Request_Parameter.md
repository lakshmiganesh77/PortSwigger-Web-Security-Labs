# Broken Access Control - Lab #7: User ID Controlled by Request Parameter

**YouTube Tutorial:** [Broken Access Control - Lab #7 User ID controlled by request parameter](https://youtu.be/-VaKuNDMkR8?si=iDGgYVlSMimFiPcN)

---

## 1. What is User ID Controlled by Request Parameter?

### Core Concept

This lab shifts from **vertical** privilege escalation (regular user → admin, Labs #3, #4, #6) to **horizontal** privilege escalation — accessing *another user's* data at the same privilege level. <cite index="107-1">This lab has a horizontal privilege escalation vulnerability on the user account page.</cite> When you visit your own account page, the application identifies which account to display using a plain, predictable `id` parameter straight from the URL — with no check that the `id` actually belongs to the currently authenticated session. Simply editing that parameter to another username pulls up their account data instead.

```
            YOUR OWN ACCOUNT                            SOMEONE ELSE'S ACCOUNT
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  GET /my-account?id=wiener  │          │  GET /my-account?id=carlos  │
   │  Cookie: [wiener's session] │          │  Cookie: [wiener's session]  │
   └─────────────┬──────────────┘          └─────────────┬──────────────┘
                 ▼                                       ▼
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  Server looks up "wiener"    │          │  Server looks up "carlos"    │
   │  → your own account details  │          │  → carlos's account details  │
   │  (matches your session)      │          │  (server never checked that  │
   └────────────────────────────┘          │  "carlos" ≠ your session!)   │
                                             └─────────────┬──────────────┘
                                                          ▼
                                          ┌────────────────────────────┐
                                          │  Response includes carlos's  │
                                          │  API key = Broken Access     │
                                          │  Control (Horizontal) ✅     │
                                          └────────────────────────────┘
```

**The `id` query parameter is the oracle** — the server trusts it completely to decide *whose* data to return, instead of deriving the account from the authenticated session.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** <cite index="107-1">This lab has a horizontal privilege escalation vulnerability on the user account page.</cite>
* **Vulnerable Parameter:** <cite index="107-1">The URL contains your username in the "id" parameter</cite> when viewing `/my-account`.
* **Broken Access Control Mechanism:** The server fetches and returns account data based purely on the `id` value supplied in the request, without verifying that `id` corresponds to the currently logged-in session.
* **Detection Channel:** Directly observing the `id` parameter in the account page's own URL, then simply substituting a different username.
* **Provided Credentials:** `wiener:peter`.
* **End Goal:** <cite index="107-1">Obtain the API key for the user carlos and submit it as the solution.</cite>

### Root Cause & Impact

* **Root Cause:** This is a textbook **Insecure Direct Object Reference (IDOR)** — the application exposes a direct reference (the `id`/username) to an internal object (a user account) and performs the lookup without any accompanying authorization check confirming the requester is allowed to view that specific object.
* **Impact:** Any authenticated user can view any other user's account data — here, the target is an API key, but in real applications this pattern frequently exposes far more sensitive information (personal details, order history, private messages) for every other user on the platform.

---

## 3. Attack Methods & Techniques (4-Phase Analysis)

* **Phase 1 — Log In and Inspect the Account URL:** Log in with the provided credentials and observe how the account page's URL identifies the current user.
* **Phase 2 — Capture the Account Request:** Send the account-page request to Burp Repeater to make it easy to modify and resend.
* **Phase 3 — Substitute the Target Username:** Change the `id` parameter's value from `wiener` to `carlos`.
* **Phase 4 — Retrieve and Submit the Leaked API Key:** Read `carlos`'s API key out of the response and submit it as the lab's solution.

### Server Behavior

* **`GET /my-account?id=wiener` (as wiener):** Returns `wiener`'s own account page, including their own API key — the expected, legitimate behavior.
* **`GET /my-account?id=carlos` (still as wiener):** Returns `carlos`'s account page and API key just as readily — the server performs no ownership check between the authenticated session and the requested `id`.

---

## 4. Step-by-Step Walkthrough

### Step 1 — Phase 1: Log In and Inspect the Account URL

1. <cite index="107-1">Log in using the supplied credentials and go to your account page.</cite>
2. **Observation:** <cite index="107-1">The URL contains your username in the "id" parameter,</cite> e.g.:

```http
GET /my-account?id=wiener HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[WIENER-SESSION]
```

3. The account page displays your own username, email, and an API key field.

### Step 2 — Phase 2: Capture the Account Request

1. With Burp Proxy intercepting, capture the `GET /my-account?id=wiener` request.
2. <cite index="107-1">Send the request to Burp Repeater.</cite>

### Step 3 — Phase 3: Substitute the Target Username

1. <cite index="107-1">Change the "id" parameter to carlos:</cite>

```http
GET /my-account?id=carlos HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[WIENER-SESSION]
```

2. Send the request.
3. **Result:** The response now renders `carlos`'s account page — while still authenticated as `wiener` — confirming the horizontal access control bypass.

### Step 4 — Phase 4: Retrieve and Submit the Leaked API Key

1. In the response body, locate `carlos`'s API key field.
2. <cite index="101-1">Copy this API key.</cite>
3. Go to the lab's **"Submit solution"** option and paste the API key as the answer.
4. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **This is the simplest form of IDOR in the series — recognize the pattern:** Whenever an object (a user, order, document, etc.) is looked up using a value the client supplies directly, always test substituting a different, plausible value for that identifier while keeping your own authenticated session.
* **Compare with the harder variant, Lab #8:** This lab uses a predictable identifier (a plain username), so guessing another user's `id` is trivial. A closely related follow-up lab uses unpredictable **GUIDs** instead, requiring you to first *discover* the target's ID elsewhere in the application (e.g. via a blog post they authored) before you can substitute it — same underlying flaw, but the identifier itself isn't guessable.
* **Always check the response, not just the status code:** A `200 OK` alone doesn't confirm success — read the actual response body to verify it truly contains `carlos`'s data (username, API key) rather than, say, an error message or your own data re-rendered.
* **No Burp Pro required:** This lab is fully solvable with **Burp Suite Community Edition** — Repeater alone is sufficient to modify and resend the request.
* **The vulnerable backend pattern:**

```javascript
// ❌ VULNERABLE — looks up whatever account the client asks for,
// with no check that it matches the authenticated session
app.get('/my-account', (req, res) => {
  const user = db.getUserByUsername(req.query.id);
  res.render('account', { user });
});

// ✅ SAFE — derive the account from the authenticated session itself,
// ignoring any client-supplied identifier for "which account is mine"
app.get('/my-account', (req, res) => {
  const user = db.getUserByUsername(req.session.username);
  res.render('account', { user });
});
```

* **Prevention summary:**
  * Never trust a client-supplied identifier to determine whose data to return — derive "the current user" from the authenticated session, not from request parameters.
  * If an endpoint must accept an identifier for a *different* resource (e.g. viewing another user's public profile), explicitly verify the requester is authorized to access that specific resource before returning it.
  * Use unpredictable identifiers (like GUIDs) as defense-in-depth, but never as a substitute for a real authorization check — see Lab #8 for why obscurity alone still fails.
  * Apply consistent object-level authorization checks across every endpoint that accepts a resource identifier, not just the most obviously sensitive ones.