# Campus Help Desk CTF: Penetration Test Write-up

| | |
|---|---|
| **Target** | Local Docker lab, `http://127.0.0.1:8080` (image `geleideng/simple-ctf:v1.0`) |
| **Installation ID** | `47c98140fc84` |
| **Scope** | Only the local lab on 127.0.0.1. No other systems were tested. |
| **Tools** | Postman (requests, cookie jar, saved examples), browser, an LLM chatbot to talk through the reasoning |
| **Evidence** | `Campus Help Desk CTF.postman_collection.json` (every request with its saved response), `CURL_STEPS.md` (the same steps as curl commands with real outputs) and screenshots in `images/` |
| **Result** | Final flag recovered (see [Flag](#flag)) |

## Summary

Three weaknesses chained together to get from an anonymous visitor to a private recovery document:

```
robots.txt ──► /backups/helpdesk.txt ──► trainee login ──► /api/notes?id=2 ──► /download?file=../private/…
 (recon)        (leaked credentials)     (valid session)    (IDOR: staff note)   (path traversal: flag)
```

| # | Finding | Severity | Impact |
|---|---|---|---|
| 1 | Credentials exposed in a backup file listed in `robots.txt` | High | Anyone can sign in as the trainee |
| 2 | Insecure direct object reference (IDOR) on `/api/notes` | High | Any signed-in user can read other users' notes, including the staff archive key |
| 3 | Path traversal in the `file` parameter of `/download` | High | Files outside the public download directory can be read |

---

## Finding 1: Credentials exposed through `robots.txt` and a backup file

**Observation.** Before logging in, `/notes` and `/documents` return 401 and show the trainee login page. I checked the standard discovery files. `/sitemap.xml` gave nothing useful, but `/robots.txt` returned:

```
User-agent: *
Disallow: /backups/helpdesk.txt
```

![robots.txt lists a disallowed URL](images/robots.txt%20lists%20disallowed%20url.png)

**Hypothesis.** `robots.txt` only asks crawlers to stay away. It doesn't block access, so the listed file is probably still public and probably contains something the developers wanted hidden.

**Test.** `GET {{baseUrl}}/backups/helpdesk.txt` without a session.

**Result.** `200 OK`. The file is an old deployment note that contains working trainee credentials:

```
username: trainee
password: trainee-ba0b1f4a59107c56c5edfc7957e95829
login: /login
```

![The disallowed URL contained a trainee login](images/disallowed%20URL%20contained%20a%20trainee%20login.png)

To confirm the login check works normally, I first sent a wrong login (`admin` / `admin`), which returned **401** "Username or password was not accepted". Then I sent the leaked credentials as `x-www-form-urlencoded` to `POST /login`. That returned **303** with a session cookie, which redirects to *My notes* ("Signed in as trainee").

![Failed login](images/Failed%20login.png)
![Successful login](images/successful%20login.png)

**Impact.** Anyone who reads `robots.txt` can sign in as the trainee.

**Mitigation.** Remove backup and deployment files from the web root. Never store credentials in plain-text notes. Disable or rotate the test account. Don't treat `robots.txt` as access control.

---

## Finding 2: IDOR on `/api/notes`

**Observation.** The *My notes* page links "Read my note" to `/api/notes?id=1`, which returns JSON:

```json
{
  "id": 1,
  "owner": "trainee",
  "body": "Orientation request ORI-2042: tomorrow's welcome pack is locked. Your trainee note has been escalated. Staff keeps the next entry in the notes system."
}
```

![A suggestive URL for IDOR](images/suggestive%20URL%20for%20IDOR.png)

**Hypothesis.** The record is chosen by a plain number in the URL, and the note says staff keeps "the next entry". If the server checks only *that* I'm logged in and not *whether I own the note*, changing `id` will return someone else's note.

**Test.** While signed in as `trainee`, I changed only the `id` value. `id=0` returned **404** "Note not found". `id=2` (the "next entry") returned:

```json
{
  "id": 2,
  "owner": "staff",
  "body": "Staff escalation for ORI-2042. The orientation archive needs the access key below. The recovery document is private; ordinary downloads start in the public directory.",
  "archive_key": "92a56d0d30510472028863ad1c510a5b",
  "recovery_file": "orientation-recovery-39a5975b500b5e4c.txt"
}
```

![id=2 has an archive key](images/id%3D2%20has%20an%20archive%20key.png)

**Result.** `200 OK` with `"owner": "staff"` while signed in as `trainee`. This confirms the server doesn't check ownership. (An early attempt returned 401 because the session cookie wasn't being sent. A 401 means "not logged in" and isn't an ownership check.)

**Impact.** Any signed-in user can read every note by counting through IDs. Here that leaked the staff archive key and the private recovery filename.

**Mitigation.** On every request, check server-side that the note's owner matches the session user, and return 403 or 404 otherwise. Unguessable IDs make enumeration harder but don't replace the ownership check.

---

## Finding 3: Path traversal on `/download`

**Observation.** `/documents` has a GET form that sends `key` and `file` to `/download` (default `file=welcome.txt`). With the archive key, the baseline request returned:

```
Orientation archive: public downloads are read from the public/ directory.
Recovery documents are stored in its sibling private/ directory.
The staff escalation note records the current recovery document name.
```

![Download with welcome.txt](images/download%20with%20welcome.txt.png)

Then I changed one thing at a time to see what the endpoint checks:

| Request (one change from the baseline) | Response | What it shows |
|---|---|---|
| `key=wrong`, `file=welcome.txt` | **403** "Archive access requires the key recorded in the staff escalation note." | The archive key is checked |
| Valid key and file, **no session cookie** | **401** (login page) | A login is required |
| `file=orientation-recovery-39a5975b500b5e4c.txt` | **404** "Document not found." | The filename is looked up inside `public/` only |

So the endpoint checks *who* is asking and *whether they have the key*, but the 404 suggests it doesn't control *where* the file path leads beyond prefixing `public/`.

**Hypothesis.** The server builds the path as `public/` + `file` without checking the result. Since `private/` is a sibling of `public/`, `../private/<name>` should resolve to the private file.

**Test.** Same key, with only `file` changed:

```
GET /download?key=92a56d0d30510472028863ad1c510a5b&file=../private/orientation-recovery-39a5975b500b5e4c.txt
```

**Result.** `200 OK` with the recovery document and the flag.

![Download with a changed directory](images/download%20with%20changed%20directory.png)

**Impact.** The login and key checks pass, and the path is still never checked. Any signed-in user with the archive key (which Finding 2 leaks to every user) can read files outside the public directory, which defeats the public/private separation.

**Mitigation.** Resolve the full path and reject the request if it's not inside `public/`. Better still, don't accept filenames: map document IDs to files on the server. Run the service with the smallest file-system access it needs.

---

## Flag

```
CTF{final_47c98140fc84fbd9bf08b307933d4b4b_ebcabfd8faa97b7eb19f25a2919c6b09}
```

Format check: it starts with `CTF{final_`, has two 32-character lowercase hex parts and ends with `}`.

## How the LLM was used

The chatbot worked as a reasoning partner, not as the testing tool. I used it to:
- Interpret clues (`robots.txt` isn't access control; "next entry" points to the next ID; "sibling directory" means `../`).
- Understand why a request returned **401** (missing session cookie) and how that differs from an ownership check (403/404).
- Learn how to recreate the browser's form login in Postman (`x-www-form-urlencoded`, cookie jar) and how to organise requests and evidence into a collection.

I sent every request myself and checked each suggestion against what the lab actually returned.

## Reproducing

1. `docker compose up -d`, then import `Campus Help Desk CTF.postman_collection.json` into Postman.
2. Set the collection variable `baseUrl` to `http://127.0.0.1:8080`. The export saves it empty.
3. Run the folders in order: `01 Recon` → `02 Login` → `03 Notes / IDOR` → `04 Documents`.
4. `docker compose down` when finished.

To reproduce without Postman, follow `CURL_STEPS.md`.
