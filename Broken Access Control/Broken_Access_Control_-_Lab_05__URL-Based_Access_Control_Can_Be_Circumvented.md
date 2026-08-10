# Broken Access Control - Lab #5: URL-Based Access Control Can Be Circumvented

**YouTube Tutorial:** [Broken Access Control - Lab #5 URL-based access control can be circumvented](https://youtu.be/AwvQrdE1Wtc?si=DPzJ-TJ_2mYURFoB)

---

## 1. What is URL-Based Access Control That Can Be Circumvented?

### Core Concept

This lab introduces a two-tier architecture that's common in real deployments: a **front-end system** (reverse proxy / load balancer / gateway) sits in front of a **back-end application server**. <cite index="88-1">This website has an unauthenticated admin panel at /admin, but a front-end system has been configured to block external access to that path.</cite> The mistake is that the front-end enforces access control purely by inspecting the *request-line URL* — it never accounts for the fact that <cite index="88-1">the back-end application is built on a framework that supports the X-Original-URL header,</cite> a header normally meant for internal routing/logging that the back-end trusts as the "real" path. By requesting an innocuous path at the front-end while smuggling the real target inside that header, an attacker routes straight past the block.

```
            BLOCKED DIRECT REQUEST                     HEADER SMUGGLING BYPASS
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  GET /admin                 │          │  GET /                      │
   │  (front-end sees "/admin"   │          │  X-Original-URL: /admin     │
   │  in the request line)       │          │  (front-end sees only "/")  │
   └─────────────┬──────────────┘          └─────────────┬──────────────┘
                 ▼                                       ▼
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  Front-end: BLOCKED          │          │  Front-end: ALLOWED          │
   │  "Access Denied"             │          │  (path looks harmless)       │
   └────────────────────────────┘          └─────────────┬──────────────┘
                                                          ▼
                                          ┌────────────────────────────┐
                                          │  Back-end reads the          │
                                          │  X-Original-URL header and   │
                                          │  routes to /admin instead    │
                                          │  = Broken Access Control ✅  │
                                          └────────────────────────────┘
```

**The mismatch between what the front-end inspects and what the back-end actually routes on is the oracle** — the front-end never sees `/admin` in the request line, but the back-end obeys the header regardless.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** <cite index="88-1">An unauthenticated admin panel at /admin, but a front-end system has been configured to block external access to that path.</cite>
* **Vulnerable Mechanism:** <cite index="88-1">The back-end application is built on a framework that supports the X-Original-URL header</cite> and trusts it for internal routing, while the front-end's access control decision is based solely on the literal path in the request line.
* **Detection Channel:** Sending a request with the path set to `/` and the `X-Original-URL` header set to `/admin` — if the admin panel loads, the back-end is routing on the header rather than the actual request path.
* **End Goal:** <cite index="88-1">Access the admin panel and delete the user carlos.</cite>

### Root Cause & Impact

* **Root Cause:** Access control is enforced at the wrong layer, and the two layers disagree about what "the URL" even is: the front-end checks the request-line path, while the back-end trusts a spoofable header to determine actual routing. As a general write-up on this class of bug puts it, <cite index="83-1">this lab is a practical demonstration of broken access control, where access validation is based on untrusted request metadata, such as custom headers, rather than server-side user context.</cite>
* **Impact:** Any restriction enforced only by a front-end's path inspection can be bypassed if the back-end honors a client-supplied override header — turning what looked like a solid perimeter block into no protection at all for the back-end's real functionality.

---

## 3. Attack Methods & Techniques (4-Phase Analysis)

* **Phase 1 — Confirm the Front-End Block:** Request `/admin` directly and confirm it's denied, noting anything about the response that hints at a front-end system.
* **Phase 2 — Test the X-Original-URL Header:** Send a request for `/` with `X-Original-URL: /admin` added, to see whether the back-end honors the header instead of the front-end-approved path.
* **Phase 3 — Access the Admin Panel via the Header:** Confirm the admin panel now loads, and locate the delete action for `carlos`.
* **Phase 4 — Apply the Same Bypass to the Delete Action:** Repeat the header-smuggling technique on the delete request itself, since it's also blocked by the same front-end restriction.

### Server Behavior

* **`GET /admin` (direct request):** Blocked by the front-end with a generic, plain "access denied" response — <cite index="84-1">attempting to access it immediately returns an Access Denied (403) response,</cite> and the response's simplicity suggests it originates from the front-end rather than the application itself.
* **`GET /` with `X-Original-URL: /admin`:** Front-end allows the request through (path looks like `/`), but the back-end reads the header and serves the admin panel content.
* **Delete action via direct path:** Also blocked by the front end for the same reason as `/admin`.
* **Delete action via `X-Original-URL` + relocated query string:** Succeeds, since it follows the same bypass pattern as accessing the panel itself.

---

## 4. Step-by-Step Walkthrough

### Step 1 — Phase 1: Confirm the Front-End Block

1. Log in or browse to the lab and click on **Admin panel** in the navigation, or request it directly:

```http
GET /admin HTTP/1.1
Host: [LAB-ID].web-security-academy.net
```

2. **Observation:** The response is a plain "access denied" message. <cite index="86-1">Trying to access the Admin Panel displays the text "access denied."</cite> The simplicity of this response (compared to the rest of the styled application) is a hint that a front-end system, not the actual application, is producing it.

### Step 2 — Phase 2: Test the X-Original-URL Header

1. Send the blocked `/admin` request to **Burp Repeater**.
2. Change the request-line path to `/` (something the front-end will allow), and add the `X-Original-URL` header pointing at the real target:

```http
GET / HTTP/1.1
Host: [LAB-ID].web-security-academy.net
X-Original-URL: /admin
```

3. Send the request.
4. **Result:** <cite index="81-1">Changing the request line to / and adding the X-Original-URL header set to /admin successfully returns the admin panel</cite> — confirming the back-end framework reads this header to determine the actual route, regardless of what the front-end saw.

### Step 3 — Phase 3: Access the Admin Panel via the Header

1. Review the rendered response (or use Burp's **Render** view / **Request in browser**) to confirm the full admin panel content, including the list of users `wiener` and `carlos`, is present.
2. Locate the delete action associated with `carlos` in the rendered HTML — this reveals the URL/parameters the delete button would normally submit (e.g. `/admin/delete?username=carlos`).

### Step 4 — Phase 4: Apply the Same Bypass to the Delete Action

1. A direct request to the delete path is blocked by the front-end the same way `/admin` was:

```http
GET /admin/delete?username=carlos HTTP/1.1
Host: [LAB-ID].web-security-academy.net
```

2. Apply the same header-smuggling technique, but this time keep the query string on the request line (moving only the path into the header):

```http
GET /?username=carlos HTTP/1.1
Host: [LAB-ID].web-security-academy.net
X-Original-URL: /admin/delete
```

3. If this returns an error about a missing parameter, it means the back-end isn't finding `username` where it expects — <cite index="86-1">this indicates the server did not read the query string from the X-Original-URL value, so the query string needs to stay on the actual request line</cite> while only the path portion moves into the header, exactly as shown above.
4. **Result:** <cite index="86-1">The response displays a 302 Found pointing to /admin, indicating the delete action succeeded and carlos has been removed.</cite>
5. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **Header placement matters in Burp:** As one write-up notes, <cite index="81-1">if you add the X-Original-URL header on the very last line before the body, Burp Suite's request editor can behave oddly — insert it as a normal header line above any body content instead.</cite>
* **Only the path goes in the header — the query string usually stays on the request line:** This lab specifically illustrates that `X-Original-URL` overrides the *path*, not necessarily the full URL with query parameters. If a parameter like `username` goes missing after the swap, try leaving it on the actual request line (e.g. `GET /?username=carlos`) while only the path moves into the header.
* **Other headers to try in real-world testing:** Some back-end frameworks respond to `X-Rewrite-URL` instead of (or in addition to) `X-Original-URL` — <cite index="84-1">poorly implemented URL-based access control can be bypassed using HTTP headers like X-Original-URL or X-Rewrite-URL,</cite> so if one doesn't work, try the other.
* **A plain/minimal denial response is a strong clue:** If a blocked page's error response looks noticeably different in style from the rest of the application (e.g. no CSS, generic wording), that's often a sign the block is happening at a separate front-end layer rather than inside the application itself — worth probing for exactly this class of bypass.
* **The vulnerable architecture pattern:**

```nginx
# ❌ VULNERABLE — front-end blocks based on the literal request path only
location /admin {
    deny all;
}
```

```javascript
// ❌ VULNERABLE — back-end framework trusts a client-supplied header for routing
const path = req.headers['x-original-url'] || req.path;
route(path);

// ✅ SAFE — never let a client-controlled header override routing or
// access-control decisions; enforce authorization in the application itself,
// not just at a perimeter layer that can be routed around
```

* **Prevention summary:**
  * Never rely solely on a front-end/perimeter layer to enforce access control — the application itself must independently verify authorization for every sensitive route.
  * Strip or ignore client-supplied routing-override headers (`X-Original-URL`, `X-Rewrite-URL`, etc.) at the trust boundary unless they're strictly required and validated.
  * Ensure front-end and back-end systems agree on what "the URL" means — any component making a security decision must use the same value the rest of the system will actually act on.
  * Apply access control consistently across every action tied to a protected resource (viewing, editing, deleting), not just the primary landing page.