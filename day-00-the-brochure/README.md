# Day 00

- **Challenge name:** [The Brochure](https://tryhackme.com/room/hh-thebrochure-081f3e36)
- **Category:** OSINT <!-- OSINT / Web / Cloud / Forensics / AI Prompt Attacks / etc. -->
- **Difficulty:** Easy
- **Date completed:** 25th of July 2026

## Summary

This challenge is about reviewing the downloadable task file - in this case, a Byte Lotus Hotel brochure saved as a `.png` file. The clues to finding the flag are hidden somewhere in the brochure.

## Exploitation / Walkthrough

### Step 1

After opening the brochure, my initial thought was to zoom in and look for a watermark or any other clue. Finding nothing, I checked the file's properties to see if anything unusual would show up there. Needless to say, I came up short.

### Step 2

Then I remembered that this challenge is about **OSINT**, and specifically about using Social Media & Username Intelligence (**SOCMINT**) techniques, as I would later find out. Looking at the brochure again, I noticed a clue that read: *"Find us on Instagram... or not."*

I opened Instagram and searched for **Byte Lotus Resort**, and found an account called **thebytelotusresort**. The account had 2 pictures uploaded as of 25th of July, but what caught my eye was that it followed exactly 1 other account, called *veratheconcierge*.

<img width="1080" height="729" alt="Zrzut ekranu 2026-07-25 215856" src="https://github.com/user-attachments/assets/9f70c0f2-45a1-48d8-a194-77409bae70bd" />

### Step 3
<img width="567" height="154" alt="Zrzut ekranu 2026-07-25 220253" src="https://github.com/user-attachments/assets/59af8cc1-b456-4237-b6bb-2290befa58fc" />

I opened the account **veratheconcierge**, visible in the image above. This account had 3 posts, each containing a string of upper- and lower-case letters, numbers, and symbols:

- `VEhNe1YzckBzX2FD`
- `QzB1bnRfaDRzX2Iz`
- `M25fZjB1bmQhfQ==`

### Step 4
These three strings are Base64-encoded. Decoding each part and joining them together gives the flag:
- `VEhNe1YzckBzX2FD` = ![redacted](https://img.shields.io/badge/-REDACTED-000000)
- `QzB1bnRfaDRzX2Iz` = ![redacted](https://img.shields.io/badge/-REDACTED-000000)
- `M25fZjB1bmQhfQ==` = ![redacted](https://img.shields.io/badge/-REDACTED-000000)

> **Note:** For decoding I used ([base64decode.org](https://www.base64decode.org/))

And voilà, there you have it!

## Flag 
![redacted](https://img.shields.io/badge/-REDACTED-000000) - to avoid spoilers, correct flag will be posted after the event is concluded.


## Lessons Learned

Key takeaway: OSINT is a powerful reminder that even a seemingly harmless marketing brochure can leak enough information (a stray hint, a linked social account) to lead straight to sensitive data. Always check the "small stuff" - social media follows, image metadata, and hidden text - before assuming a source is clean.
