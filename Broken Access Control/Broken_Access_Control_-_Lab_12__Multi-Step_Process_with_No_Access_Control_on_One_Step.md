# Broken Access Control - Lab #12: Multi-Step Process with No Access Control on One Step

**YouTube Tutorial:** [Broken Access Control - Lab #12 Multi-step process with no access control on one step](https://youtu.be/XhNzjdT0Azk?si=3Ftr377_mxNU5nle)

---

## 1. What is a Multi-Step Process with No Access Control on One Step?

### Core Concept

<cite index="138-1">This lab has an admin panel with a flawed multi-step process for changing a user's role.</cite> Promoting a user to administrator isn't a single request — it's a **two-step workflow**: an initial "Upgrade user" action, followed by a "Are you sure?" confirmation step that actually applies the change. The developers correctly protected the **first** step with an authorization check, but seem to have assumed that if you can't reach step one, you'll never reach step two either. In reality, both steps are independent HTTP requests, and only one of them is actually guarded — the confirmation request can be replayed on its own by anyone, entirely bypassing the first (protected) step.

```
            LEGITIMATE ADMIN FLOW                       ATTACKER'S BYPASS
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  STEP 1 (protected):         │          │  Attacker never touches      │
   │  POST /admin-roles            │          │  Step 1 at all — skips it     │
   │  username=carlos&action=      │          │  entirely                    │
   │  upgrade                      │          │                                │
   │  → checked: is caller admin?  │          └─────────────┬──────────────┘
   └─────────────┬──────────────┘                        ▼
                 ▼                          ┌────────────────────────────┐
   ┌────────────────────────────┐          │  STEP 2 (UNPROTECTED):        │
   │  Server responds with a       │          │  Confirmation request,        │
   │  confirmation prompt          │          │  replayed directly with       │
   │  "Are you sure?"               │          │  attacker's own username      │
   └─────────────┬──────────────┘          │  and a non-admin session        │
                 ▼                          └─────────────┬──────────────┘
   ┌────────────────────────────┐                        ▼
   │  STEP 2: confirmed, role       │          ┌────────────────────────────┐
   │  actually changes               │          │  No auth check here —        │
   └────────────────────────────┘          │  role change succeeds anyway  │
                                             │  = Broken Access Control ✅  │
                                             └────────────────────────────┘
```

**The confirmation request is the oracle** — the developer protected the door but forgot to lock the room behind it; going straight to the confirmation step, with your own username and your own unprivileged session, is enough.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** <cite index="138-1">An admin panel with a flawed multi-step process for changing a user's role.</cite>
* **Familiarization Credentials:** <cite index="138-1">You can familiarize yourself with the admin panel by logging in using the credentials administrator:admin.</cite>
* **Target Credentials:** <cite index="138-1">Log in using the credentials wiener:peter and exploit the flawed access controls to promote yourself to become an administrator.</cite>
* **Broken Access Control Mechanism:** The workflow's **initial** request (`POST /admin-roles`) enforces an authorization check, but the **confirmation** request that follows it does not — an attacker can send the confirmation directly, with arbitrary parameters, under any session.
* **Detection Channel:** Capturing both steps of the legitimate admin flow in Burp, then testing whether the second step alone succeeds when replayed under a non-admin session.
* **End Goal:** Self-promote the `wiener` account to administrator using only the confirmation step, without ever using the administrator's own session.

### Root Cause & Impact

* **Root Cause:** Access control was applied inconsistently across a multi-step process — the developer assumed reaching the confirmation step implied the (protected) first step had already been legitimately completed, rather than independently verifying authorization on every step of the workflow.
* **Impact:** As one summary of this vulnerability class puts it, <cite index="133-1">access controls are enforced on early steps but skipped on later steps under the assumption the user followed the full process,</cite> which lets an attacker bypass the protected steps entirely and jump directly to the unprotected one to perform the sensitive action.

---

## 3. Attack Methods & Techniques (4-Phase Analysis)

* **Phase 1 — Familiarize with the Multi-Step Admin Flow:** Log in as `administrator:admin` and walk through promoting a test user, noting that two separate requests are involved.
* **Phase 2 — Capture Both Requests:** Intercept the initial upgrade request and the subsequent confirmation request in Burp, sending the confirmation to Repeater.
* **Phase 3 — Swap in the Non-Admin Session and Target:** Log in separately as `wiener`, copy that session cookie into the confirmation request, and change the target username to `wiener`.
* **Phase 4 — Replay the Confirmation Step Alone:** Send the modified confirmation request under `wiener`'s session, bypassing the (protected) first step entirely.

### Server Behavior

* **Step 1 (`POST /admin-roles`, initial upgrade) as a non-admin session:** Correctly denied — the authorization check on this specific step works as intended.
* **Step 2 (confirmation request) as the administrator, following step 1 normally:** Succeeds — this is the legitimate flow, useful for observing the confirmation request's exact shape.
* **Step 2 (confirmation request) sent directly, under `wiener`'s non-admin session, with `username=wiener`:** Also succeeds — <cite index="131-1">the application implements access controls on the initial request, but not the confirmation request,</cite> so the role change is applied with no authorization check at all on this step.

---

## 4. Step-by-Step Walkthrough

### Step 1 — Phase 1: Familiarize with the Multi-Step Admin Flow

1. Log in using <cite index="138-1">administrator:admin.</cite>
2. Navigate to the **admin panel**, where <cite index="130-1">the administrator can upgrade or downgrade a user's privilege.</cite>
3. Click **Upgrade user** next to `carlos` (a safe test target) and observe that the application first shows a confirmation prompt — <cite index="137-1">"Are you sure?"</cite> — before the change is actually applied. This confirms the process genuinely has two distinct steps.

### Step 2 — Phase 2: Capture Both Requests

1. With Burp Proxy intercepting, click **Upgrade user** for `carlos` again.
2. Observe the first request:

```http
POST /admin-roles HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[ADMINISTRATOR-SESSION]

username=carlos&action=upgrade
```

3. Forward it, then confirm the action on the resulting "Are you sure?" prompt while still intercepting.
4. <cite index="132-1">Send the confirmation HTTP request to Burp Repeater.</cite> This second request is the one that actually applies the role change.

### Step 3 — Phase 3: Swap in the Non-Admin Session and Target

1. <cite index="132-1">Open a private/incognito browser window, and log in with the non-admin credentials</cite> `wiener:peter`.
2. <cite index="132-1">Copy the non-admin user's session cookie</cite> from this new login.
3. In the Repeater tab holding the confirmation request, replace the `Cookie` header's session value with `wiener`'s session, and <cite index="132-1">change the username to yours</cite> (`wiener`):

```http
POST /admin-roles-confirm HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[WIENER-SESSION]

username=wiener&action=upgrade
```

### Step 4 — Phase 4: Replay the Confirmation Step Alone

1. Send this modified confirmation request — note that `wiener`'s session never touched Step 1 at all in this attempt.
2. <cite index="131-1">The response returns a 302, indicating the request succeeded</cite> — `wiener` has been promoted to administrator using only the unprotected confirmation step.
3. Log back in as `wiener` and confirm the admin panel is now accessible.
4. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **The goal is self-escalation via wiener's own session, not just proving the confirmation step is broken:** As with Lab #6, make sure the final successful request used `wiener`'s session throughout — the point is that `wiener`, who never had access to the protected first step, still ends up an administrator.
* **Multi-step and multi-stage workflows deserve extra scrutiny in general:** As the vulnerability class is described, <cite index="133-1">an attacker can bypass earlier steps and directly send a crafted request to a later step to perform the action without proper authorization</cite> — whenever an application splits a sensitive action into multiple requests (confirmations, wizards, review-then-submit flows), test each step's authorization independently rather than assuming the whole flow is protected just because the entry point is.
* **Compare with Lab #6:** Both labs achieve vertical self-escalation by finding an alternate, unprotected path to the same underlying action — Lab #6 exploited an HTTP *method* the check didn't cover; this lab exploits a *step* in a workflow the check didn't cover. Same root cause (inconsistent authorization coverage), different specific gap.
* **Confirmation/"Are you sure?" steps are a common blind spot:** Developers often treat a confirmation step as "just UI," forgetting it's a real, independently-reachable endpoint that needs its own authorization check.
* **The vulnerable backend pattern:**

```javascript
// ❌ VULNERABLE — authorization only checked on the initial request;
// the confirmation endpoint trusts that step 1 already happened
app.post('/admin-roles', requireAdmin, (req, res) => {
  res.render('confirm-role-change', { username: req.body.username });
});
app.post('/admin-roles-confirm', (req, res) => { // no requireAdmin here!
  changeUserRole(req.body.username, req.body.action);
});

// ✅ SAFE — apply the same authorization check to every step of the workflow
app.post('/admin-roles', requireAdmin, (req, res) => {
  res.render('confirm-role-change', { username: req.body.username });
});
app.post('/admin-roles-confirm', requireAdmin, (req, res) => {
  changeUserRole(req.body.username, req.body.action);
});
```

* **Prevention summary:**
  * Enforce authorization checks independently on every step of a multi-step process — never assume a later step is only reachable after an earlier, protected step has already run.
  * Treat confirmation/review endpoints as first-class, independently-attackable requests during security testing, not as trivial UI formalities.
  * Where possible, bind multi-step workflows together with a server-side, single-use state token so a later step genuinely cannot be replayed or reached out of order — but pair this with authorization checks on every step, not as a substitute for them.
  * Regularly test each individual request in a multi-step flow in isolation, using tools like Burp Repeater, rather than only testing the flow as a whole through the UI.