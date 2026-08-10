# Broken Access Control - Lab #10: User ID Controlled by Request Parameter with Password Disclosure

**YouTube Tutorial:** [Broken Access Control - Lab #10 User ID controlled by request parameter with password disclosure](https://youtu.be/tcPkT82pa6k?si=tDrPdGFJUdwHq-1k)

---

## 1. What is Data Leakage of a Password via ID Swapping?

### Core Concept

This lab combines two flaws from earlier labs into a full chain, ending in vertical rather than horizontal escalation. First, <cite index="109-1">this lab has a user account page that contains the current user's existing password, prefilled in a masked input.</cite> On its own that's a minor issue — a masked `<input>` still has the real password sitting in its `value` attribute in the HTML source, visible to anyone who inspects the page. Second, exactly like Labs #7–#9, the account page is fetched via the same vulnerable `id` parameter. Chain the two together, and swapping `id` from your own username to `administrator` doesn't just leak someone else's account details — it leaks their **actual password in plaintext**, which you can then use to log in for real.

```
            NORMAL BEHAVIOR                              CHAINED EXPLOIT
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  GET /my-account?id=wiener  │          │  GET /my-account?id=        │
   │  Password field: ●●●●●●●●●● │          │  administrator                │
   │  (masked in the rendered UI)│          │  (still your own session)    │
   └─────────────┬──────────────┘          └─────────────┬──────────────┘
                 ▼                                       ▼
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  View source: masked field   │          │  View source: masked field   │
   │  actually contains "peter"   │          │  actually contains the       │
   │  in plaintext in the HTML    │          │  ADMINISTRATOR's real        │
   └────────────────────────────┘          │  plaintext password!         │
                                             └─────────────┬──────────────┘
                                                          ▼
                                          ┌────────────────────────────┐
                                          │  Log out, log back in as     │
                                          │  administrator using the     │
                                          │  leaked password = full        │
                                          │  admin account takeover ✅   │
                                          └────────────────────────────┘
```

**The masked password field's HTML source is the oracle** — the visual mask hides it from a casual glance, but it was never actually protected data-wise; combined with the `id` swap, it discloses another user's real credentials outright.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** <cite index="109-1">A user account page that contains the current user's existing password, prefilled in a masked input.</cite>
* **Vulnerable Parameter:** The same `id` parameter on `/my-account?id=...` as Labs #7–#9.
* **Broken Access Control Mechanism:** Two flaws combined — (1) no ownership check on the `id` parameter (same as before), and (2) the account page unnecessarily embeds the real, plaintext password value in the HTML, relying purely on visual masking (`type="password"`) rather than never sending the value to the client at all.
* **Target Identifier:** Unlike Labs #7–#9, the target here isn't `carlos` — it's the `administrator` account itself.
* **Provided Credentials:** `wiener:peter`.
* **End Goal:** <cite index="109-1">Retrieve the administrator's password, then use it to delete the user carlos.</cite>

### Root Cause & Impact

* **Root Cause:** The application conflates *visual* masking with *actual* data protection. A masked `<input type="password">` still ships the real value to the client in the page source — combined with the same missing ownership check on `id` seen throughout this module, this lets any authenticated user retrieve any other user's literal password.
* **Impact:** This is more severe than Labs #7–#9: instead of leaking an API key scoped to a specific feature, it leaks a real, reusable login credential. Given that the target is `administrator`, exploiting this achieves full vertical privilege escalation — not just viewing another user's data, but logging in and acting as them entirely.

---

## 3. Attack Methods & Techniques (4-Phase Analysis)

* **Phase 1 — Log In and Inspect the Password Field:** Log in as `wiener`, view your own account page, and confirm the masked password field's HTML source reveals your real password.
* **Phase 2 — Swap the id Parameter to administrator:** Change the `id` parameter to `administrator`, exactly as in Labs #7–#9's IDOR pattern.
* **Phase 3 — Extract the Administrator's Plaintext Password:** View the resulting page's source to read the administrator's real password out of the masked field's value.
* **Phase 4 — Log In as Administrator and Delete Carlos:** Log out, log back in with the leaked administrator credentials, and use the now-legitimate admin session to delete `carlos`.

### Server Behavior

* **`GET /my-account?id=wiener` (as wiener):** Returns your own account page; the password `<input>` visually shows dots, but its underlying `value` attribute contains your actual password (`peter`) in plaintext.
* **`GET /my-account?id=administrator` (still as wiener):** Returns the administrator's account page — same IDOR as Labs #7–#9 — with the password field's `value` now containing the administrator's real plaintext password.
* **Login with leaked administrator credentials:** Succeeds normally, since it's now a genuinely valid (leaked) credential pair, not a forged cookie or role field.

---

## 4. Step-by-Step Walkthrough

### Step 1 — Phase 1: Log In and Inspect the Password Field

1. <cite index="109-1">Log in using the supplied credentials and access the user account page.</cite>
2. View the page source (Ctrl+U) or inspect the password field via DevTools.
3. **Observation:** The password input displays as masked dots in the rendered page, but its HTML `value` attribute contains your real password in plain text:

```html
<input type="password" name="password" value="peter" ...>
```

This confirms the vulnerability exists in principle — masking is cosmetic only.

### Step 2 — Phase 2: Swap the id Parameter to administrator

1. <cite index="109-1">Change the "id" parameter in the URL to administrator:</cite>

```http
GET /my-account?id=administrator HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[WIENER-SESSION]
```

2. Send the request (via browser address bar edit, or Burp Repeater for a cleaner view of the raw response).

### Step 3 — Phase 3: Extract the Administrator's Plaintext Password

1. View the resulting page's source.
2. **Result:** The masked password field's `value` attribute now contains the administrator's actual password:

```html
<input type="password" name="password" value="[ADMINISTRATOR-PASSWORD]" ...>
```

3. Copy this value — this is a real, working credential, not just leaked account metadata.

### Step 4 — Phase 4: Log In as Administrator and Delete Carlos

1. Log out of the `wiener` session.
2. Log back in using:

```
Username: administrator
Password: [ADMINISTRATOR-PASSWORD]
```

3. Navigate to the admin panel (or wherever user management is available for this lab).
4. Delete the user `carlos`.
5. **Result:** <cite index="103-1">Delete carlos, and you will have solved the lab!</cite>
6. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **Visual masking is not data protection:** This is the core lesson of the lab. An HTML `type="password"` attribute only changes how a browser *renders* the field — it does nothing to prevent the raw value from being present in, and readable from, the page source. Sensitive values that don't need to round-trip to the client shouldn't be sent at all.
* **This lab chains two separate flaws — recognize both:** The IDOR (`id` parameter swap) alone, as seen in Labs #7–#9, is necessary but not sufficient here; it's the *combination* with the unnecessarily-disclosed password field that turns a data leak into full credential theft.
* **Always view source on prefilled form fields during testing:** Any form that pre-populates a field with what looks like masked/sensitive data (passwords, tokens, keys) is worth checking in the raw HTML — a masked *appearance* says nothing about what's actually being transmitted.
* **This is the first lab in the series with true vertical escalation via leaked credentials:** Labs #3, #4, and #6 escalated privilege through cookie forgery, mass assignment, and method swapping respectively. This lab achieves the same vertical outcome through information disclosure instead — a good reminder that privilege escalation doesn't always require bypassing a check directly; sometimes it just requires the check to leak something it shouldn't.
* **The vulnerable backend pattern:**

```html
<!-- ❌ VULNERABLE — the real password value is sent to the client,
     masking is purely a rendering hint, not actual protection -->
<input type="password" name="password" value="{{ user.password }}">

<!-- ✅ SAFE — never send the actual password back to the client at all;
     leave the field empty and let the user re-enter it only if they want to change it -->
<input type="password" name="password" value="" placeholder="••••••••">
```

* **Prevention summary:**
  * Never pre-populate a password field (or any sensitive secret) with its real value — leave it blank, or use a non-reversible placeholder if a "password is set" indicator is needed.
  * Fix the underlying IDOR on `id`-based endpoints regardless of what data they might expose — this lab shows how severity escalates dramatically depending on what happens to be sitting behind the missing check.
  * Treat any client-rendered masking (passwords, censored account numbers, etc.) as a UX feature only, never a substitute for actually withholding sensitive data server-side.
  * Regularly audit account/profile pages specifically for accidental inclusion of secrets in HTML source, not just in visible page content.