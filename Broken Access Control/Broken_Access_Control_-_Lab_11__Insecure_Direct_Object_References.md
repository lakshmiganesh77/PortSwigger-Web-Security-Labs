# Broken Access Control - Lab #11: Insecure Direct Object References

**YouTube Tutorial:** [Broken Access Control - Lab #11 Insecure direct object references](https://youtu.be/EaMWR5Cmjkg?si=RzctRZ03M0pS-Ob_)

---

## 1. What is Insecure Direct Object References (IDOR)?

### Core Concept

This lab formally names the vulnerability class that Labs #7–#10 were all specific examples of. <cite index="129-1">IDOR vulnerabilities often arise when sensitive resources are located in static files on the server-side filesystem.</cite> <cite index="126-1">This lab stores user chat logs directly on the server's file system, and retrieves them using static URLs.</cite> Every chat transcript is saved as a simple incrementing filename (`1.txt`, `2.txt`, `3.txt`, ...) with no ownership check on retrieval — so once you download your own transcript and see its number, decrementing it lets you read a completely different user's private conversation, including anything sensitive they shared, like a password.

```
            YOUR OWN TRANSCRIPT                          SOMEONE ELSE'S TRANSCRIPT
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  Live chat → send a message  │          │  Same download endpoint,     │
   │  → View transcript           │          │  just a different filename   │
   │  → downloads "2.txt"         │          │                                │
   └─────────────┬──────────────┘          └─────────────┬──────────────┘
                 ▼                                       ▼
   ┌────────────────────────────┐          ┌────────────────────────────┐
   │  GET /download-transcript/   │          │  GET /download-transcript/   │
   │  2.txt                       │          │  1.txt                       │
   │  → your own chat history     │          │  (guessed — one lower)       │
   └────────────────────────────┘          └─────────────┬──────────────┘
                                                          ▼
                                          ┌────────────────────────────┐
                                          │  Response contains a          │
                                          │  DIFFERENT user's chat log,   │
                                          │  including a leaked password  │
                                          │  = Broken Access Control ✅  │
                                          └────────────────────────────┘
```

**The predictable, sequential filename is the oracle** — no session, cookie, or token is checked at all; possessing the right filename is treated as sufficient proof of ownership.

---

## 2. Attack Overview & Key Concepts

### Vulnerability Explanation

* **Vulnerable Feature:** <cite index="126-1">This lab stores user chat logs directly on the server's file system, and retrieves them using static URLs.</cite>
* **Vulnerable Endpoint:** `GET /download-transcript/<N>.txt`, where `<N>` is a plain, sequential, guessable integer.
* **Broken Access Control Mechanism:** The endpoint performs no authorization check whatsoever — any transcript filename returns its contents to any requester, authenticated or not, regardless of who created it.
* **Detection Channel:** <cite index="126-1">Select the Live chat tab. Send a message and then select View transcript</cite> to observe your own transcript's filename, then simply try a lower number.
* **End Goal:** <cite index="126-1">Solve the lab by finding the password for the user carlos, and logging into their account.</cite>

### Root Cause & Impact

* **Root Cause:** As the PortSwigger reference material summarizes, <cite index="129-1">a website might save chat message transcripts to disk using an incrementing filename, and allow users to retrieve these by visiting a URL</cite> constructed from that filename, without any accompanying server-side check that the requester is entitled to view that specific file.
* **Impact:** Because the transcripts contain freeform user input — including, in this case, a password a user typed into the chat — an attacker can enumerate every transcript on the system and harvest whatever sensitive information other users have inadvertently shared, potentially leading to full account takeover.

---

## 3. Attack Methods & Techniques (4-Phase Analysis)

* **Phase 1 — Trigger a Transcript and Note Its Filename:** Use the live chat feature, download your own transcript, and observe the numeric filename pattern.
* **Phase 2 — Capture the Download Request:** Intercept the `GET /download-transcript/<N>.txt` request in Burp to make it easy to modify.
* **Phase 3 — Enumerate Lower-Numbered Transcripts:** Decrement the filename (starting from `1.txt`) and resend, reading each response for anything sensitive.
* **Phase 4 — Extract the Password and Log In as Carlos:** Locate a transcript containing a password, then use it to log in as `carlos`.

### Server Behavior

* **`GET /download-transcript/<your-number>.txt`:** Returns your own chat transcript, as expected.
* **`GET /download-transcript/<lower-number>.txt`:** Returns a completely different user's transcript with no error, no access-denied response, and no authentication check of any kind.

---

## 4. Step-by-Step Walkthrough

### Step 1 — Phase 1: Trigger a Transcript and Note Its Filename

1. <cite index="126-1">Select the Live chat tab. Send a message</cite> to the chat bot (any content works).
2. <cite index="126-1">Select View transcript.</cite>
3. **Observation:** <cite index="124-1">Clicking "Download Transcript" downloads a file such as 2.txt</cite> — note that this number may already be higher than `1`, meaning at least one earlier transcript exists on the server.

### Step 2 — Phase 2: Capture the Download Request

1. With Burp Proxy intercepting (or via **Proxy → HTTP History**), locate the request:

```http
GET /download-transcript/2.txt HTTP/1.1
Host: [LAB-ID].web-security-academy.net
```

2. <cite index="124-1">Right-click the request and send it to Repeater.</cite>

### Step 3 — Phase 3: Enumerate Lower-Numbered Transcripts

1. In Repeater, change the filename to a lower number, starting with the lowest plausible value:

```http
GET /download-transcript/1.txt HTTP/1.1
Host: [LAB-ID].web-security-academy.net
```

2. Send the request.
3. **Result:** <cite index="123-1">The response is not your own chat transcript — it belongs to a different conversation entirely,</cite> confirming the IDOR: no ownership check exists on this endpoint.

### Step 4 — Phase 4: Extract the Password and Log In as Carlos

1. Read through the leaked transcript's contents.
2. **Result:** <cite index="121-1">The file contains a password, which may belong to the user carlos.</cite>
3. Navigate to the login page and log in with:

```
Username: carlos
Password: [PASSWORD FROM 1.txt]
```

4. **Result:** <cite index="124-1">The login succeeds, confirming the password belonged to carlos.</cite>
5. The lab banner updates to **"Congratulations, you solved the lab!"**.

---

## 5. Pro-Tips & Common Pitfalls

* **This is the "textbook" IDOR the whole preceding sequence was building toward:** Labs #7–#10 all showed variations of missing ownership checks on user-supplied identifiers; this lab makes it explicit and names the vulnerability class directly, using a filesystem-backed resource instead of a database-backed user account.
* **Always start numbering from 1, not just "one less than yours":** If your own transcript came back as, say, `4.txt`, don't assume `3.txt` is the only other transcript worth checking — enumerate from the lowest plausible number, since multiple prior sessions (including the target's) may exist.
* **Predictable, sequential identifiers are a red flag anywhere they appear:** File names, database IDs, order numbers, ticket references — any resource identified by a simple auto-incrementing value is trivial to enumerate once you have one valid example and no ownership check is enforced.
* **No Burp Pro required:** As one write-up notes, <cite index="127-1">this lab requires only Burp Suite Community Edition</cite> — Repeater alone (or even just editing the download URL directly in a browser) is enough to solve it.
* **The vulnerable backend pattern:**

```javascript
// ❌ VULNERABLE — serves any transcript file by name, with no ownership check
app.get('/download-transcript/:filename', (req, res) => {
  res.sendFile(`/chats/${req.params.filename}`);
});

// ✅ SAFE — verify the requested transcript actually belongs to the
// authenticated session before serving it
app.get('/download-transcript/:filename', (req, res) => {
  const transcript = db.getTranscriptByFilename(req.params.filename);
  if (transcript.ownerId !== req.session.userId) {
    return res.status(403).send('Forbidden');
  }
  res.sendFile(transcript.path);
});
```

* **Prevention summary:**
  * Never use predictable, sequential identifiers (like incrementing filenames) for sensitive per-user resources — use unguessable, randomly generated identifiers as a baseline mitigation.
  * Always enforce a server-side ownership/authorization check before serving any resource tied to a specific user, regardless of how the resource is stored (database record or flat file).
  * Avoid storing sensitive user-submitted content (like passwords typed into a chat) in plaintext, logged, or easily retrievable form in the first place.
  * Apply the same access control discipline to auxiliary features like live chat, file downloads, and support transcripts as to the application's primary account and admin functionality.