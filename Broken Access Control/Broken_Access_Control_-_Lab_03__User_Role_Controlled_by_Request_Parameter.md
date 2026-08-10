# Broken Access Control - Lab #3: User Role Controlled by Request Parameter

**YouTube Tutorial:** [Broken Access Control - Lab #3 User role controlled by request parameter](https://youtu.be/e_jsPdEeSto?si=ypaqaUGdplScXq1k)

---

## 1. What is User Role Controlled by Request Parameter?

### Core Concept

Labs #1 and #2 relied on *finding* an unprotected admin URL. This lab is different: the admin route itself performs a check — but that check is based on a **client-controlled cookie** the server trusts blindly instead of verifying server-side. <cite index="69-1">This lab has an admin panel at /admin, which identifies administrators using a forgeable cookie.</cite> Since the client fully controls its own cookies, an attacker can simply flip the value and grant themselves administrator status without ever compromising a real admin account.

```
            LEGITIMATE LOGIN (wiener)                  ATTACKER'S MANIPULATION
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  Login as wiener:peter      │          │  Same login, but intercept  │
   │  Server sets:               │          │  the response in Burp       │
   │  Cookie: Admin=false        │          │  Proxy before it's saved     │
   └─────────────┬──────────────┘          └─────────────┬──────────────┘
                 ▼                                       ▼
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  GET /admin                 │          │  Edit the cookie value:     │
   │  Cookie: Admin=false        │          │  Admin=false → Admin=true   │
   │  → Access denied            │          │  before forwarding it        │
   └────────────────────────────┘          └─────────────┬──────────────┘
                                                          ▼
                                          ┌────────────────────────────┐
                                          │  GET /admin                 │
                                          │  Cookie: Admin=true          │
                                          │  → Full admin panel access ✅│
                                          └────────────────────────────┘
```

**The `Admin` cookie value is the oracle** — the server never re-verifies the user's actual role against any trusted, server-side source; it just reads whatever the client claims.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** <cite index="69-1">An admin panel at /admin, which identifies administrators using a forgeable cookie.</cite>
* **Vulnerable Mechanism:** The `Admin` cookie set on login (e.g. `Admin=false`) is trusted directly by the server to decide access — with no corresponding server-side session record confirming the user's actual role.
* **Detection Channel:** Intercepting the login response in Burp Proxy reveals the cookie being set, showing exactly which value controls access.
* **Provided Credentials:** <cite index="69-1">You can log in to your own account using the following credentials: wiener:peter.</cite>
* **End Goal:** <cite index="69-1">Solve the lab by accessing the admin panel and using it to delete the user carlos.</cite>

### Root Cause & Impact

* **Root Cause:** Authorization state (whether the current user is an administrator) is stored in a client-side, user-editable cookie instead of being derived server-side from an authenticated session tied to a trusted role record in the backend.
* **Impact:** Any authenticated (or even unauthenticated, depending on implementation) user can trivially self-escalate to administrator by editing a single cookie value — no credential theft, no session hijacking, no exploitation of a separate bug required.

---

## 3. Attack Methods & Techniques (4-Phase Analysis)

* **Phase 1 — Confirm the Admin Panel Is Protected:** Browse to `/admin` while unauthenticated (or as a normal user) and confirm access is denied, establishing a baseline.
* **Phase 2 — Log In and Inspect the Role Cookie:** Log in with the provided credentials, intercepting the login response to see the `Admin` cookie being set to `false`.
* **Phase 3 — Forge the Admin Cookie:** Modify the `Admin` cookie value from `false` to `true`, either by editing the intercepted response or via browser DevTools/Repeater on subsequent requests.
* **Phase 4 — Access the Panel and Delete the User:** Reload the admin panel with the forged cookie and use it to delete `carlos`.

### Server Behavior

* **Unauthenticated or normal-user request to `/admin`:** Returns an access-denied message (e.g. "Admin interface only available if logged in as an administrator").
* **Request with `Admin=false` cookie:** Same access-denied response — the logged-in `wiener` account is not a genuine administrator.
* **Request with `Admin=true` cookie (forged):** Returns the full admin panel — the server trusted the cookie's claim without any further verification.

---

## 4. Step-by-Step Walkthrough

### Step 1 — Phase 1: Confirm the Admin Panel Is Protected

1. <cite index="69-1">Browse to /admin and observe that you can't access the admin panel.</cite>
2. Note the exact wording of the denial message — this confirms some check is happening, just not a robust one.

```http
GET /admin HTTP/1.1
Host: [LAB-ID].web-security-academy.net
```

### Step 2 — Phase 2: Log In and Inspect the Role Cookie

1. <cite index="69-1">Browse to the login page. In Burp Proxy, turn interception on and enable response interception.</cite>
2. <cite index="69-1">Complete and submit the login page, and forward the resulting request in Burp</cite> using the provided credentials `wiener:peter`.
3. With response interception on, inspect the `Set-Cookie` headers in the login response.
4. **Observation:** A cookie such as `Admin=false` is set alongside the session cookie — this is the value the server (incorrectly) trusts to determine administrator status.

```http
HTTP/1.1 302 Found
Set-Cookie: session=[SESSION-VALUE]
Set-Cookie: Admin=false
```

### Step 3 — Phase 3: Forge the Admin Cookie

1. While still intercepting the response, edit the cookie value directly:

```http
Set-Cookie: Admin=true
```

2. Forward the modified response so the browser stores `Admin=true` instead of `Admin=false`.
3. Alternatively, use browser DevTools (**Application → Cookies**) to edit the stored cookie value after login, or edit the `Cookie` header on subsequent requests directly in Burp Repeater.

### Step 4 — Phase 4: Access the Panel and Delete the User

1. Reload `/admin` (or click **"My account" → Admin panel** if now visible) with the forged cookie in place:

```http
GET /admin HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[SESSION-VALUE]; Admin=true
```

2. **Result:** The admin panel now loads successfully, listing users including `wiener` and `carlos`.
3. Click **Delete** next to `carlos`. Ensure the `Admin=true` cookie is also present on this delete request (intercept and forward/edit it in Burp if it isn't automatically included).

```http
GET /admin/delete?username=carlos HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[SESSION-VALUE]; Admin=true
```

4. **Result:** The user `carlos` is deleted successfully.
5. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **Check every request, not just the first one:** The `Admin` cookie must be present and set to `true` on **every** privileged request, including the final delete action — some walkthroughs get tripped up because the delete request doesn't carry the forged cookie automatically and needs its own edit.
* **Enable response interception specifically:** By default, Burp Proxy intercepts requests, not responses. You need to explicitly turn on response interception (**Proxy → Options → Intercept Server Responses**) to catch and edit the `Set-Cookie` header before the browser stores it.
* **DevTools works just as well as Burp for this step:** Since the cookie is client-readable and editable, you can skip Burp entirely for Step 3 and just edit the cookie value directly in your browser's **Application/Storage** tab — Burp is more convenient for chaining this with the login/delete requests, but not required.
* **This is different from a session/JWT-based role claim:** Here the role is stored as a plain, separate cookie value with no cryptographic protection (no signature, no server-side lookup) — contrast this with later labs in the module that use tamper-evident tokens, which require a different bypass technique entirely.
* **The vulnerable backend pattern:**

```php
// ❌ VULNERABLE — trusts a client-supplied cookie to determine authorization
if ($_COOKIE['Admin'] === 'true') {
    renderAdminPanel();
}

// ✅ SAFE — derive role from a trusted, server-side session/database lookup
$user = getAuthenticatedUser($session);
if ($user && $user->role === 'admin') {
    renderAdminPanel();
}
```

* **Prevention summary:**
  * Never store authorization decisions (roles, permissions) in a value the client can read or modify, such as a plain cookie.
  * Derive the user's role server-side on every request, from a trusted session store or database, not from client-supplied data.
  * If any client-visible token must carry role information, use a signed/encrypted format (e.g. a properly validated JWT) and verify its signature server-side on every use.
  * Apply access control checks consistently across all related actions (viewing the panel, deleting a user, etc.), not just the initial page load.