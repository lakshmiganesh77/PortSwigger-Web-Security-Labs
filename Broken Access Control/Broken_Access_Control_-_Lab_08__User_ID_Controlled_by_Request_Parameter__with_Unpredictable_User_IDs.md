# Broken Access Control - Lab #8: User ID Controlled by Request Parameter, with Unpredictable User IDs

**YouTube Tutorial:** [Broken Access Control - Lab #8 User ID controlled by request parameter, with unpredictable user IDs](https://youtu.be/aaIfsH-fP5c?si=bQaRLxKjK8DHhipl)

---

## 1. What is User ID Controlled by Request Parameter, with Unpredictable User IDs?

### Core Concept

This lab is the direct sequel to Lab #7, and it demonstrates exactly why unpredictable identifiers are only a partial mitigation, not a real fix. <cite index="108-1">This lab has a horizontal privilege escalation vulnerability on the user account page, but identifies users with GUIDs</cite> instead of plain usernames. You can no longer just type `carlos` into the `id` parameter — a GUID like `55b2eb8a-6a18-40f9-bd35-7c04b4939bae` <cite index="100-1">isn't something you can guess.</cite> But the underlying flaw is identical: the server still trusts whatever `id` value the client supplies, with no check that it belongs to the requester. The only extra step is **finding** the target's GUID somewhere else in the application first — and it turns out the app leaks it freely.

```
            LAB #7 (predictable ID)                    LAB #8 (unpredictable ID)
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  id=carlos                  │          │  id=55b2eb8a-6a18-40f9-      │
   │  (guessable directly)        │          │  bd35-7c04b4939bae           │
   └────────────────────────────┘          │  (must be discovered first)  │
                                             └─────────────┬──────────────┘
                                                          ▼
                                          ┌────────────────────────────┐
                                          │  Browse the blog, find a     │
                                          │  post authored by carlos     │
                                          └─────────────┬──────────────┘
                                                          ▼
                                          ┌────────────────────────────┐
                                          │  The post's own request/     │
                                          │  response discloses carlos's │
                                          │  GUID as his user ID          │
                                          └─────────────┬──────────────┘
                                                          ▼
                                          ┌────────────────────────────┐
                                          │  GET /my-account?id=<GUID>  │
                                          │  = carlos's data leaks       │
                                          │  anyway — same flaw ✅       │
                                          └────────────────────────────┘
```

**A blog post authored by the target becomes the new oracle** — the identifier is unguessable, but the application discloses it anyway through an unrelated feature.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** <cite index="108-1">This lab has a horizontal privilege escalation vulnerability on the user account page, but identifies users with GUIDs.</cite>
* **Vulnerable Parameter:** The same `id` parameter as Lab #7 on `/my-account`, except its expected value is now a GUID rather than a username.
* **Broken Access Control Mechanism:** Identical to Lab #7 — the server returns account data for whatever `id` is supplied, without verifying it matches the authenticated session. The only difference is the extra reconnaissance step required to obtain a valid target ID.
* **Detection/Discovery Channel:** <cite index="108-1">Find a blog post by carlos. Click on carlos and observe that the URL contains his user ID.</cite>
* **Provided Credentials:** `wiener:peter`.
* **End Goal:** <cite index="108-1">Find the GUID for carlos, then submit his API key as the solution.</cite>

### Root Cause & Impact

* **Root Cause:** Same IDOR as Lab #7 — no server-side check that the requested `id` belongs to the current session — combined with a secondary information disclosure: the application exposes user GUIDs through an unrelated feature (blog post author links), undermining the unpredictability that was meant to be a safeguard.
* **Impact:** This lab is a direct demonstration that **unpredictable identifiers are not equivalent to access control**. If an unguessable ID can be discovered through any other legitimate application feature (as GUIDs so often are — profile links, authored content, activity feeds), the "unpredictability" defense collapses entirely, and the underlying missing authorization check is exploited exactly as easily as in Lab #7.

---

## 3. Attack Methods & Techniques (4-Phase Analysis)

* **Phase 1 — Log In and Confirm the GUID-Based ID:** Log in and observe that your own account page now uses a GUID, not a plain username, in the `id` parameter.
* **Phase 2 — Find Carlos's GUID via a Blog Post:** Browse the site's blog/posts section, locate one authored by `carlos`, and extract his GUID from the resulting request or link.
* **Phase 3 — Substitute the Discovered GUID:** Replace your own `id` value with `carlos`'s GUID in the account-page request.
* **Phase 4 — Retrieve and Submit the Leaked API Key:** Read `carlos`'s API key from the response and submit it as the solution.

### Server Behavior

* **`GET /my-account?id=<your-GUID>` (as wiener):** Returns your own account details, including your own API key — normal, expected behavior.
* **Viewing a post authored by carlos:** The page or its underlying request discloses `carlos`'s user ID as a GUID, typically as part of a link to his profile or author page.
* **`GET /my-account?id=<carlos's-GUID>` (still as wiener):** Returns `carlos`'s account details and API key — the server performs no ownership check, exactly as in Lab #7.

---

## 4. Step-by-Step Walkthrough

### Step 1 — Phase 1: Log In and Confirm the GUID-Based ID

1. Log in using the provided credentials `wiener:peter`.
2. <cite index="108-1">Access your account page</cite> and inspect the URL:

```http
GET /my-account?id=55b2eb8a-6a18-40f9-bd35-7c04b4939bae HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[WIENER-SESSION]
```

3. **Observation:** Unlike Lab #7, the `id` value is a long, random GUID rather than a plain username — <cite index="100-1">this time, it's using a GUID (Globally Unique Identifier)</cite> that can't realistically be guessed.

### Step 2 — Phase 2: Find Carlos's GUID via a Blog Post

1. <cite index="108-1">Find a blog post by carlos.</cite> Browse the site's homepage or blog listing and look for an author byline reading "carlos".
2. <cite index="108-1">Click on carlos and observe that the URL contains his user ID.</cite> This is often a link from the post to an author/profile view.
3. **Result:** Capturing this request in Burp (or simply reading the resulting URL/response) reveals `carlos`'s GUID, e.g.:

```
f26a0928-06ae-4b0d-be0a-ca03266160f0
```

4. <cite index="108-1">Make a note of this ID</cite> for use in the next step.

### Step 3 — Phase 3: Substitute the Discovered GUID

1. Send your `/my-account` request to **Burp Repeater**.
2. <cite index="108-1">Change the "id" parameter to the saved user ID</cite> for `carlos`:

```http
GET /my-account?id=f26a0928-06ae-4b0d-be0a-ca03266160f0 HTTP/1.1
Host: [LAB-ID].web-security-academy.net
Cookie: session=[WIENER-SESSION]
```

3. Send the request.
4. **Result:** The response now renders `carlos`'s account page — still authenticated as `wiener` — confirming the same horizontal access control bypass as Lab #7, just requiring one extra discovery step.

### Step 4 — Phase 4: Retrieve and Submit the Leaked API Key

1. In the response body, locate `carlos`'s API key field.
2. Copy the API key.
3. Go to the lab's **"Submit solution"** option and paste the API key as the answer.
4. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **Unpredictability is not authorization — this lab exists to prove that point:** GUIDs stop casual guessing, but they don't stop anything once the ID is disclosed through some other feature. Always ask "where else in this application might this identifier leak?" rather than assuming an unguessable ID is automatically safe.
* **Look for identifiers in author/profile links, not just the account page itself:** Blogs, comments, activity feeds, "shared by" labels, and public profile pages are common places where a supposedly private identifier gets exposed as a side effect of an unrelated feature.
* **Check both the rendered link and the underlying request:** Sometimes the GUID is visible directly in an `href`; other times it only appears in the request Burp captures when you click through (e.g. an API call the page makes after loading) — check both the page source and your HTTP history.
* **Compare directly with Lab #7:** The vulnerability class, the vulnerable endpoint, and the exploitation technique (swap the `id` parameter) are identical. The only real difference is the extra reconnaissance step to first obtain a value that can't simply be typed in from memory.
* **The vulnerable backend pattern:**

```javascript
// ❌ VULNERABLE — GUID makes guessing hard, but there's still no
// check that the requested id belongs to the authenticated session
app.get('/my-account', (req, res) => {
  const user = db.getUserByGuid(req.query.id);
  res.render('account', { user });
});

// ❌ ALSO VULNERABLE — leaking the GUID anywhere defeats the "unpredictability" defense
app.get('/post', (req, res) => {
  const author = db.getUserByUsername(post.author);
  res.render('post', { post, authorId: author.guid }); // leaked!
});

// ✅ SAFE — derive the account from the authenticated session itself,
// and never expose internal object identifiers unnecessarily
app.get('/my-account', (req, res) => {
  const user = db.getUserBySession(req.session);
  res.render('account', { user });
});
```

* **Prevention summary:**
  * Never rely on unpredictable identifiers (GUIDs, UUIDs, random tokens) as a substitute for real, server-side authorization checks — treat them purely as identifiers, not as secrets.
  * Audit every feature that displays or links to another user's identifier (author bylines, profile links, shared content) for unintended disclosure.
  * Derive "the current user" from the authenticated session for any endpoint returning account-specific data, regardless of what identifier the client supplies.
  * Apply the same object-level authorization check used for predictable IDs (Lab #7) consistently, even when the identifier scheme is changed to something harder to guess.