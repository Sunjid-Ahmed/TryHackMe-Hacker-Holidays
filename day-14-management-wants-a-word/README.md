# Day 14 - Management Wants a Word

- **Challenge:** [Management Wants a Word (Hacker Holidays, Day 14 - final day)](https://tryhackme.com/room/hh-managementwantsaword-6bf3cc41)
- **Category:** Forensics
- **Difficulty:** Hard
- **Date completed:** 9th of August 2026

## Summary

A guest named Vera left her laptop behind after an early checkout. IT pulled a full KAPE triage before wiping it. The task: dig through the artifacts, find a password she never meant to leave behind, and see where it leads.

The whole thing turned out to be a chain: crack her Windows login from a leftover LSA secret, use that to unlock her DPAPI-protected data, use that to unlock a saved Chrome password, and use that password to open a hidden VeraCrypt container sitting in her Documents folder.

## Exploitation / Walkthrough

### Step 1 - Dumping the local hashes and secrets

The KAPE package included the `SAM`, `SYSTEM`, and `SECURITY` registry hives. Impacket's `secretsdump.py` can parse these offline, no live Windows box needed:

```
python3 secretsdump.py -sam SAM -system SYSTEM -security SECURITY LOCAL
```

Buried in the LSA secrets was this:

```
[*] DefaultPassword
(Unknown User):minivera
```

`DefaultPassword` is a leftover from Windows autologon configuration - it stores the plaintext password. That's Vera's Windows account password: `minivera`.

### Step 2 - Cracking open her DPAPI masterkey

Windows uses DPAPI to protect a lot of user secrets (saved browser passwords, Wi-Fi keys, etc.), and each user has a "masterkey" that's itself protected by their login password. I found Vera's masterkey file under:

```
AppData/Roaming/Microsoft/Protect/<her SID>/c90719ef-5b98-474e-b934-136d606a702a
```

Using `dpapick3` and the password from Step 1, the masterkey decrypted cleanly.

### Step 3 - Unlocking her saved Chrome password

Chrome encrypts its saved logins with an AES key, and that AES key is itself wrapped with DPAPI (stored in the `Local State` file). With the masterkey in hand, I decrypted that AES key, then used it to decrypt her `Login Data` SQLite database.

There was exactly one saved login:

```
url:  http://bytelotus.thm:8080/
user: VeraSecretVault
pass: Wh4t1sV3raD0inG0nTh1sH0st
```

Given @0xMia's story hint ("a browser will remember things for you that you never told anyone else"), this was clearly the thing to chase.

### Step 4 - Finding the hidden container

There was a 100MB file called `backup` sitting in Vera's Documents folder, full of what looked like random bytes - no readable header, no file signature. That's the classic signature of a VeraCrypt container (they're designed to look like random noise so nobody can prove they exist).

The sandbox I was working in didn't have access to the kernel crypto module that `cryptsetup` needs, so mounting it the normal way wasn't an option. Instead I wrote my own AES-XTS decryptor in Python and used it to decrypt the VeraCrypt header directly with the password from Step 3 (standard VeraCrypt defaults: PBKDF2-HMAC-SHA512, 500,000 iterations). It decrypted clean on the first proper attempt - the `VERA` magic bytes and checksum lined up.

### Step 5 - Walking the filesystem by hand

Inside the container was a FAT32 filesystem. With no way to mount it, I parsed the FAT32 structures manually (boot sector, FAT table, directory entries) to walk the folder tree and pull files out sector by sector using the same custom decryptor. That turned up a folder called `secret_financial_documents` with a CSV of transactions and a PDF invoice.

### Step 6 - The invoice

Rendering the extracted PDF showed a totally normal-looking invoice from Byte Lotus Resorts - except line item #1 read:

```
Flag: REDACTED
```

And voila, there you have it!

## Flag

![REDACTED](https://img.shields.io/badge/flag-REDACTED-black)

The correct flag will be posted once the event concludes, to avoid spoilers.

## Lessons Learned

- `DefaultPassword` in LSA secrets is an easy win if it's there - always check it before trying to crack hashes.
- DPAPI is a chain: password -> masterkey -> app-specific key -> actual secret. Each step needs the one before it.
- A file that's just random-looking bytes with no header is worth treating as an encrypted container, not junk data.
- When the "proper" tooling isn't available (no kernel module for `cryptsetup` here), it's still possible to implement the crypto by hand if you understand the format - AES-XTS and FAT32 aren't magic, just spec-following.
- A really good "hidden" file was hiding in the most obvious place: a saved browser password, no cracking required.
