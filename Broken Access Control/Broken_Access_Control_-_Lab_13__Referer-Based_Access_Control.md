# Broken Access Control - Lab #13: Referer-Based Access Control

**YouTube Tutorial:** [Broken Access Control - Lab #13 Referer-based access control](https://youtu.be/DjZ_WHlYOq0?si=QYKiZiy1M6SOYFZE)

---

## 1. What is Referer-Based Access Control?

### Core Concept

This final lab in the module protects admin functionality with a check on the `Referer` HTTP header — <cite index="145-1">this lab controls access to certain admin functionality based on the Referer header.</cite> The idea, presumably, is that only requests originating from a link on the admin panel page itself should be allowed to trigger the sensitive action, since a browser normally sets `Referer` to the page you clicked from. But the `Referer` header is entirely client-controlled — nothing stops an attacker's own browser (or Burp Repeater) from simply setting it to whatever value the check expects, regardless of where the request is actually coming from.

```
            LEGITIMATE ADMIN CLICK                       ATTACKER'S FORGED HEADER
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  Admin clicks "Upgrade" on   │          │  Attacker sends the exact    │
   │  the /admin panel page        │          │  same request, but manually  │
   │  Browser auto-sets:           │          │  sets:                        │
   │  Referer: .../admin           │          │  Referer: .../admin           │
   └─────────────┬──────────────┘          └─────────────┬──────────────┘
                 ▼                                       ▼
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  Server: Referer matches      │          │  Server: Referer matches      │
   │  expected value → ALLOWED     │          │  expected value → ALLOWED      │
   └────────────────────────────┘          │  (even though attacker never   │
                                             │  actually visited /admin!)     │
                                             └─────────────┬──────────────┘
                                                          ▼
                                          ┌────────────────────────────┐
                                          │  wiener promoted to admin,   │
                                          │  using wiener's own session   │
                                          │  = Broken Access Control ✅  │
                                          └────────────────────────────┘
```

**The `Referer` header value is the oracle** — and since it's just a client-supplied string like any other header, it carries zero real proof about where the request actually originated.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** <cite index="145-1">This lab controls access to certain admin functionality based on the Referer header.</cite>
* **Familiarization Credentials:** <cite index="145-1">You can familiarize yourself with the admin panel by logging in using the credentials administrator:admin.</cite>
* **Target Credentials:** <cite index="145-1">Log in using the credentials wiener:peter and exploit the flawed access controls to promote yourself to become an administrator.</cite>
* **Broken Access Control Mechanism:** The server checks whether the incoming request's `Referer` header matches the expected admin-panel URL, and treats a match as sufficient proof of legitimate origin — <cite index="142-1">if the promotion URL is directly copied and requested without that header set correctly, it prompts a 401 Unauthorized error because of the Referer check,</cite> confirming the check exists, but it can simply be added back in manually.
* **Detection Channel:** Capturing the legitimate promotion request as an administrator, noting the exact `Referer` value it sends, then replaying that same value under a non-admin session.
* **End Goal:** Self-promote `wiener` to administrator by forging the `Referer` header on an otherwise-unprivileged request.

### Root Cause & Impact

* **Root Cause:** The `Referer` header is fully attacker-controlled client metadata — browsers set it automatically under normal navigation, but any HTTP client (curl, Burp, a custom script) can set it to an arbitrary value. Using it as an access control signal conflates "the browser happened to send this header" with "the caller is actually authorized."
* **Impact:** Like the `X-Original-URL` bypass in Lab #5, this is another case of trusting client-supplied request metadata for a security decision. Once the expected value is known (trivial to observe from a single legitimate request), the check can be satisfied by anyone, regardless of their actual privilege level or navigation history.

---

## 3. Attack Methods & Techniques (4-Phase Analysis)

* **Phase 1 — Familiarize with the Admin Panel and Referer Check:** Log in as `administrator:admin`, trigger a role-promotion action, and note the `Referer` header the browser automatically attaches.
* **Phase 2 — Confirm the Referer Requirement:** Replay the promotion request without the correct `Referer` value under a non-admin session, and observe it's denied.
* **Phase 3 — Forge the Referer Header:** Add the expected `Referer` value back into the request manually.
* **Phase 4 — Replay Under wiener's Session:** Send the forged request using `wiener`'s own session and target username, confirming the promotion succeeds.

### Server Behavior

* **Promotion request as administrator, with browser-set `Referer: .../admin`:** Succeeds — the legitimate flow.
* **Promotion request as `wiener`, with no `Referer` header (or an unrelated one):** Denied with a `401 Unauthorized`, confirming the check is real and actively enforced.
* **Promotion request as `wiener`, with a manually forged `Referer: .../admin` header:** Succeeds — the check only verifies the header's *value*, not whether the request genuinely originated from that page.

---

## 4. Step-by-Step Walkthrough

### Step 1 — Phase 1: Familiarize with the Admin Panel and Referer Check

1. Log in using <cite index="145-1">administrator:admin.</cite>
2. Navigate to the admin panel and, with Burp Proxy intercepting, click **Upgrade user** for `carlos` (a safe test target).
3. Inspect the captured request and note the `Referer` header the browser attached automatically:

```http
GET /admin-roles?username=carlos&action=upgrade HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Referer: https://[LAB-ID].web-security-academy.net/admin
Cookie: session=[ADMINISTRATOR-SESSION]
```

4. Send this request to **Burp Repeater** for further testing.

### Step 2 — Phase 2: Confirm the Referer Requirement

1. Log in separately (in an incognito window) as `wiener:peter` and copy that session's cookie.
2. Swap the session in the Repeater request, but **remove or blank out** the `Referer` header:

```http
GET /admin-roles?username=wiener&action=upgrade HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[WIENER-SESSION]
```

3. Send it.
4. **Result:** <cite index="142-1">A 401 Unauthorized error is returned because of the Referer check</cite> — confirming this specific header is actively required for the action to succeed.

### Step 3 — Phase 3: Forge the Referer Header

1. Add the `Referer` header back into the request, using the exact value observed in Step 1:

```http
GET /admin-roles?username=wiener&action=upgrade HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Referer: https://[LAB-ID].web-security-academy.net/admin
Cookie: session=[WIENER-SESSION]
```

### Step 4 — Phase 4: Replay Under wiener's Session

1. Send the request with the forged `Referer` header and `wiener`'s own session, targeting `wiener` himself.
2. **Result:** The request succeeds — `wiener` is promoted to administrator, despite never actually having navigated to `/admin` in this session, and despite never using the administrator's session at all.
3. Log back in as `wiener` and confirm the admin panel is now accessible.
4. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **Confirm the check is real before bypassing it:** As in Step 2, deliberately triggering the `401 Unauthorized` first (by omitting or altering the `Referer`) is worth doing — it proves the header genuinely gates the action, so the eventual bypass demonstrates a real vulnerability rather than a coincidence.
* **Any client-supplied header is spoofable — treat all of them the same way:** This lab is functionally similar to Lab #5's `X-Original-URL` bypass. Both `Referer` and custom routing headers are just strings the client chooses to send; neither can serve as proof of anything about the request's true origin or the caller's actual privilege.
* **The goal is self-escalation via wiener's own session:** As with Labs #6 and #12, make sure the final successful request uses `wiener`'s session and targets `wiener` himself — not the administrator's session, which would trivially work but wouldn't demonstrate the actual bypass.
* **This closes out the module's theme:** Across all 13 labs, the common thread is a security decision made by trusting something the client controls — a URL, a cookie, a JSON field, an HTTP method, a workflow step, or (here) a header — instead of deriving authorization from a trusted, server-side source of truth.
* **The vulnerable backend pattern:**

```javascript
// ❌ VULNERABLE — trusts a client-supplied header as proof of legitimate origin
app.get('/admin-roles', (req, res) => {
  if (req.headers['referer'] !== `${BASE_URL}/admin`) {
    return res.status(401).send('Unauthorized');
  }
  changeUserRole(req.query.username, req.query.action);
});

// ✅ SAFE — derive authorization from the authenticated session's role,
// never from spoofable request metadata like Referer
app.get('/admin-roles', requireAdmin, (req, res) => {
  changeUserRole(req.query.username, req.query.action);
});
```

* **Prevention summary:**
  * Never use the `Referer` header (or any other client-supplied header) as an access control mechanism — it carries no cryptographic or server-verified guarantee about a request's true origin.
  * Derive authorization exclusively from a trusted, server-side session tied to a verified user role, independent of any request headers the client can freely set.
  * If origin-checking is genuinely needed (e.g. CSRF mitigation), use it only as defense-in-depth alongside — never instead of — a real authorization check.
  * Audit every admin/privileged endpoint in an application for this exact pattern: any check based on a header, cookie, or parameter the client fully controls is not actually enforcing anything.