# Broken Access Control - Lab #4: User Role Can Be Modified in User Profile

**YouTube Tutorial:** [Broken Access Control - Lab #4 User role can be modified in user profile](https://youtu.be/jjfe7WRN76o?si=9HwhF4VGEHd1fV9j)

---

## 1. What is User Role Can Be Modified in User Profile?

### Core Concept

Lab #3 forged a role that was already exposed as a separate cookie. This lab is subtler: <cite index="80-1">this lab has an admin panel at /admin. It's only accessible to logged-in users with a roleid of 2.</cite> The `roleid` field isn't shown anywhere in the UI, and there's no obvious form field for it — but the **response** from an unrelated profile action (updating your email) happens to reveal it. Because the backend accepts whatever JSON fields the client sends without restricting which ones are writable, an attacker can simply add the `roleid` field to a request that was never meant to modify it — a classic **mass assignment** flaw.

```
            LEGITIMATE EMAIL UPDATE                    ATTACKER'S ADDED FIELD
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  POST /my-account/          │          │  Same endpoint, but the     │
   │  change-email                │          │  attacker adds an extra     │
   │  { "email": "x@y.com" }      │          │  field the client never     │
   └─────────────┬──────────────┘          │  normally sends:             │
                 ▼                          │  { "email": "x@y.com",       │
   ┌────────────────────────────┐          │    "roleid": 2 }             │
   │  Response reveals a field    │          └─────────────┬──────────────┘
   │  never shown in the UI:      │                        ▼
   │  { ..., "roleid": 1 }        │          ┌────────────────────────────┐
   └────────────────────────────┘          │  Server accepts the extra    │
                                             │  field and updates roleid    │
                                             │  to 2 — no validation ✅     │
                                             └────────────────────────────┘
```

**The email-update response is the oracle** — it accidentally discloses a field name (`roleid`) that the request itself never needed to send, handing the attacker exactly what to inject.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** <cite index="80-1">An admin panel at /admin, only accessible to logged-in users with a roleid of 2.</cite>
* **Vulnerable Endpoint:** <cite index="71-1">POST /my-account/change-email</cite> — a routine profile-update action that has nothing to do with roles on the surface.
* **Broken Access Control Mechanism:** The server performs mass assignment: any field present in the submitted JSON body is written to the user's record, including sensitive fields like `roleid`, with no allow-list restricting which fields the client is permitted to set.
* **Detection Channel:** <cite index="80-1">The response to the email-update request contains your role ID</cite> — an unintentional disclosure of both the field's existence and its current value.
* **Provided Credentials:** `wiener:peter`.
* **End Goal:** <cite index="80-1">Solve the lab by accessing the admin panel and using it to delete the user carlos.</cite>

### Root Cause & Impact

* **Root Cause:** The backend deserializes the entire client-supplied JSON body directly onto the user's data model (a common pattern with ORMs/frameworks that auto-bind request bodies) without an explicit allow-list of which fields the endpoint is actually meant to update.
* **Impact:** Any authenticated user can escalate their own privileges by guessing or discovering a sensitive field name and injecting it into an unrelated, otherwise-legitimate request — no separate exploit or credential compromise needed, just a JSON body they fully control.

---

## 3. Attack Methods & Techniques (4-Phase Analysis)

* **Phase 1 — Confirm the Admin Panel Is Protected:** Log in as `wiener` and confirm `/admin` denies access to a `roleid`-1 user.
* **Phase 2 — Discover the roleid Field via the Email Update:** Intercept the email-change request/response and notice that the response leaks a `roleid` field never present in the request.
* **Phase 3 — Inject roleid into the Request Body:** Add `"roleid":2` to the change-email JSON body and resend it, observing that the server accepts and applies it.
* **Phase 4 — Access the Panel and Delete the User:** Reload the account/admin page with the escalated role and use it to delete `carlos`.

### Server Behavior

* **`/admin` with `roleid=1`:** Returns an access-denied response — the current user isn't recognized as an administrator.
* **`POST /my-account/change-email` (normal request):** Returns a JSON response confirming the update, and incidentally includes the current `roleid` value (e.g. `1`) even though the request body never mentioned it.
* **`POST /my-account/change-email` with injected `"roleid":2`:** The server accepts the extra field with no validation and updates the account's role, reflected in the response.
* **`/admin` after escalation:** Returns the full admin panel — the mass-assignment injection succeeded.

---

## 4. Step-by-Step Walkthrough

### Step 1 — Phase 1: Confirm the Admin Panel Is Protected

1. Log in using the provided credentials `wiener:peter`.
2. Browse directly to `/admin`.

```http
GET /admin HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[SESSION-VALUE]
```

3. **Observation:** Access is denied — confirming `wiener`'s current role isn't sufficient (`roleid` is not `2`).

### Step 2 — Phase 2: Discover the roleid Field via the Email Update

1. Navigate to **My account** and use the **update email** feature to submit any valid email address.
2. With Burp Proxy intercepting, capture the request:

```http
POST /my-account/change-email HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Content-Type: application/json
Cookie: session=[SESSION-VALUE]

{"email":"wiener@normal-user.net"}
```

3. Forward the request and inspect the response:

```json
{
  "username": "wiener",
  "email": "wiener@normal-user.net",
  "apikey": "Qvbkfk3gByoLDZrgkvPw43om5BsJC7nz",
  "roleid": 1
}
```

4. **Observation:** <cite index="80-1">The response contains your role ID</cite> — `roleid: 1` — even though the request never included that field. This tells you the field's exact name and confirms the endpoint's backend model includes it.

### Step 3 — Phase 3: Inject roleid into the Request Body

1. Send the `change-email` request to **Repeater**.
2. <cite index="80-1">Add "roleid":2 into the JSON in the request body, and resend it.</cite>

```http
POST /my-account/change-email HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Content-Type: application/json
Cookie: session=[SESSION-VALUE]

{"email":"wiener@normal-user.net","roleid":2}
```

3. **Result:** <cite index="80-1">The response shows your roleid has changed to 2</cite> — the server accepted and applied the injected field with no validation against what the endpoint is actually supposed to modify.

### Step 4 — Phase 4: Access the Panel and Delete the User

1. Reload your account page or browse directly to `/admin` again:

```http
GET /admin HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[SESSION-VALUE]
```

2. **Result:** The admin panel now loads, listing users including `wiener` and `carlos`.
3. Click **Delete** next to `carlos`.
4. **Result:** The user `carlos` is deleted successfully.
5. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **Watch responses, not just requests, for field disclosure:** This lab's whole trick hinges on the fact that a completely unrelated action (updating an email) leaked a sensitive field name in its response. Get in the habit of reading full JSON responses in Burp, not just skimming for the expected fields.
* **Try adding, not just modifying, fields:** Unlike Lab #3 (editing an existing cookie), this lab requires injecting a field that was never in the original request at all. If a response reveals a field name you don't control, try adding it to related write requests, even if the client-side form has no UI for it.
* **This is classic mass assignment / over-posting:** The vulnerability class here is distinct from Lab #3's simple cookie trust issue — it specifically involves a backend that binds the entire client-supplied body onto a data model, rather than only reading the specific fields it expects.
* **Content-Type matters:** Since the request body is JSON, make sure edits preserve valid JSON syntax (correct commas, matching braces) — a malformed body may cause the server to reject the whole request rather than partially apply it.
* **The vulnerable backend pattern:**

```javascript
// ❌ VULNERABLE — binds the entire request body directly onto the user model
app.post('/my-account/change-email', (req, res) => {
  Object.assign(currentUser, req.body); // roleid can be overwritten!
  currentUser.save();
  res.json(currentUser);
});

// ✅ SAFE — explicitly allow-list only the fields this endpoint should modify
app.post('/my-account/change-email', (req, res) => {
  currentUser.email = req.body.email;
  currentUser.save();
  res.json(currentUser);
});
```

* **Prevention summary:**
  * Never bind an entire client-supplied request body directly onto a data model — explicitly allow-list which fields each endpoint is permitted to modify.
  * Never return sensitive/internal fields (like a role ID) in API responses unless the client genuinely needs them.
  * Enforce authorization checks server-side for any field that affects privilege, independent of which endpoint the field arrives through.
  * Apply the same access control validation to all fields in a request body, not just the ones the client-side UI is designed to send.