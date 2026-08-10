# Broken Access Control - Lab #6: Method-Based Access Control Can Be Circumvented

**YouTube Tutorial:** [Broken Access Control - Lab #6 Method-based access control can be circumvented](https://youtu.be/0mWVYM_dqIg?si=h54gWhyAdcHhTrBO)

---

## 1. What is Method-Based Access Control That Can Be Circumvented?

### Core Concept

<cite index="95-1">This lab implements access controls based partly on the HTTP method of requests.</cite> The developers protected the "promote user" action for `POST` requests — likely with a role check applied specifically to that method — but never applied the same check when the identical action is performed with a different HTTP method, like `GET`. Since most backend frameworks will happily route a `GET` request to the same handler logic if the URL and parameters match, simply changing the verb sails straight past the access control that was only ever wired up for `POST`.

```
            POST REQUEST (protected)                  GET REQUEST (unprotected)
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  POST /admin/roles          │          │  GET /admin/roles?          │
   │  username=wiener&action=    │          │  username=wiener&action=    │
   │  upgrade                     │          │  upgrade                     │
   │  Cookie: [wiener's session]  │          │  Cookie: [wiener's session]  │
   └─────────────┬──────────────┘          └─────────────┬──────────────┘
                 ▼                                       ▼
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  Server: role check applies  │          │  Server: role check only     │
   │  to POST — wiener isn't      │          │  wired up for POST — GET     │
   │  admin → 401/403             │          │  bypasses it entirely        │
   └────────────────────────────┘          └─────────────┬──────────────┘
                                                          ▼
                                          ┌────────────────────────────┐
                                          │  302 Found — wiener is now   │
                                          │  an administrator             │
                                          │  = Broken Access Control ✅  │
                                          └────────────────────────────┘
```

**The HTTP method itself is the oracle** — identical parameters, identical endpoint, identical session — the only thing that changes between "blocked" and "allowed" is the verb.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** <cite index="95-1">This lab implements access controls based partly on the HTTP method of requests.</cite>
* **Familiarization Credentials:** <cite index="95-1">You can familiarize yourself with the admin panel by logging in using the credentials administrator:admin.</cite>
* **Target Credentials:** <cite index="91-1">Log in using the credentials wiener:peter and exploit the flawed access controls to promote yourself to become an administrator.</cite>
* **Broken Access Control Mechanism:** The role-promotion endpoint checks the caller's privilege only when the request arrives as `POST`. The exact same action performed via `GET` (with the same parameters) reaches the same underlying logic without that check.
* **Detection Channel:** Capturing the legitimate admin "upgrade user" `POST` request, then replaying it as a `GET` under a non-admin session to see if the promotion still succeeds.
* **End Goal:** Escalate the `wiener` account itself to administrator, entirely using the flawed method-based check — <cite index="96-1">the goal is to escalate yourself (as wiener) to an admin, without having access to the administrator account.</cite>

### Root Cause & Impact

* **Root Cause:** Access control logic was bound to a specific HTTP method (`POST`) rather than to the underlying action/endpoint itself. Most web frameworks route multiple HTTP methods to the same handler unless explicitly restricted, so a check applied only in the `POST` code path leaves any other method that reaches the same logic completely unguarded.
* **Impact:** Any user, regardless of actual privilege, can perform a privileged action just by changing the request method — no header spoofing, no cookie forgery, no parameter injection required, only a change in verb.

---

## 3. Attack Methods & Techniques (4-Phase Analysis)

* **Phase 1 — Familiarize with the Admin Panel:** Log in with the provided `administrator:admin` credentials to see how the legitimate "promote user" action is structured.
* **Phase 2 — Capture the Promotion Request:** Trigger the promote action on a throwaway/test target and intercept the resulting `POST` request in Burp.
* **Phase 3 — Swap Sessions and Method:** Log in as `wiener` in a separate session, substitute `wiener`'s session cookie into the captured request, and change the request's target username to `wiener` himself.
* **Phase 4 — Convert POST to GET and Resend:** Change the HTTP method from `POST` to `GET`, moving parameters into the query string, and resend under `wiener`'s session to confirm the bypass.

### Server Behavior

* **`POST /admin/roles` as `wiener` (non-admin):** Denied — the role check correctly blocks a non-administrator from calling this method.
* **`POST /admin/roles` as `administrator`:** Succeeds normally — this is the legitimate, intended flow, useful only for observing the request's shape.
* **`GET /admin/roles?...` as `wiener` (non-admin):** Succeeds despite `wiener` having no admin privileges — <cite index="90-1">the response displays a 302 Found with directory location /admin, meaning the promotion has succeeded</cite> — confirming the access control gap.

---

## 4. Step-by-Step Walkthrough

### Step 1 — Phase 1: Familiarize with the Admin Panel

1. Log in using the provided admin credentials: `administrator:admin`.
2. <cite index="98-1">Click on the admin panel and locate the option to upgrade a user's role.</cite>
3. Browse the panel to understand its layout — note the list of users and the "Upgrade" action next to each.

### Step 2 — Phase 2: Capture the Promotion Request

1. With Burp Proxy intercepting, click **Upgrade** next to a non-critical test user (or note you'll target `wiener` directly in the next steps).
2. <cite index="90-1">Intercept the upgrade-user request and send it to Repeater</cite>, then drop the original intercepted request so it doesn't actually complete yet.

```http
POST /admin/roles HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Cookie: session=[ADMINISTRATOR-SESSION]

username=wiener&action=upgrade
```

3. Log out of the administrator account — you no longer need it for the rest of the exploit.

### Step 3 — Phase 3: Swap Sessions and Method

1. <cite index="90-1">Open an incognito/private browser window and log in using the non-admin credentials wiener:peter.</cite>
2. Copy `wiener`'s session cookie value from this new login.
3. Back in the captured Repeater request, replace the `Cookie` header's session value with `wiener`'s session:

```http
POST /admin/roles HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Cookie: session=[WIENER-SESSION]

username=wiener&action=upgrade
```

4. Send this as-is first to confirm the baseline: since it's still a `POST` and `wiener` isn't an admin, this should be denied — proving the `POST` path is genuinely protected.

### Step 4 — Phase 4: Convert POST to GET and Resend

1. In Repeater, change the request method from `POST` to `GET`, and move the body parameters into the query string:

```http
GET /admin/roles?username=wiener&action=upgrade HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[WIENER-SESSION]
```

2. Send the request.
3. **Result:** <cite index="90-1">The response displays a 302 Found with directory location /admin, meaning the URL function has successfully upgraded the user wiener to admin</cite> — despite `wiener` having no administrator privileges and never having used the administrator's session.
4. Confirm by logging back in as `wiener` and observing that the admin panel is now accessible.
5. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **The lab isn't solved just by seeing "wiener is now admin":** Some walkthroughs point out a common mistake — <cite index="96-1">it's easy to see wiener is now an admin and assume you're done, but the actual goal is to have escalated yourself as wiener, without ever having used the administrator account's session</cite> to perform the promotion. Make sure the final successful request used `wiener`'s own session, not the administrator's.
* **Burp's "Change request method" feature does most of the work:** Right-click the request in Repeater and use **Change request method** to auto-convert `POST` to `GET` (and move body parameters into the query string) rather than doing it by hand — faster and less error-prone.
* **Test the POST-as-wiener baseline first:** Confirming that `POST` genuinely is blocked for `wiener` (Step 3) isolates the variable — it proves the bypass in Step 4 is really about the method change, not some other difference between the two attempts.
* **Different from Lab #5's technique:** Lab #5 exploited a mismatch between what a front-end and back-end each treated as "the URL." This lab exploits a mismatch between which HTTP methods a single back-end's own access-control logic actually covers — same broad theme (inconsistent enforcement), different specific mechanism.
* **The vulnerable backend pattern:**

```javascript
// ❌ VULNERABLE — role check only applied inside the POST handler
app.post('/admin/roles', requireAdmin, upgradeUserHandler);
app.get('/admin/roles', upgradeUserHandler); // same handler, no check!

// ✅ SAFE — apply the same authorization middleware regardless of HTTP method
app.all('/admin/roles', requireAdmin, upgradeUserHandler);
```

* **Prevention summary:**
  * Never tie access control checks to a specific HTTP method — apply them to the underlying action/route regardless of which verb reaches it.
  * Explicitly restrict sensitive endpoints to only the HTTP methods they're intended to support (e.g. reject `GET` on state-changing actions) in addition to authorization checks.
  * Centralize authorization logic (middleware, filters) so it's automatically applied to every method a route responds to, rather than being duplicated (and potentially forgotten) per method handler.
  * Regularly test sensitive endpoints with multiple HTTP methods during security reviews, not just the one the client-side UI happens to use.