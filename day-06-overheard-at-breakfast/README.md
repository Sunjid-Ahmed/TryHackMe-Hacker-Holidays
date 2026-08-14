# Day 06 - Overheard at Breakfast

- **Challenge:** [Overheard at Breakfast](https://tryhackme.com/room/hh-overheardatbreakfast-6f01793c)
- **Category:** OSINT
- **Difficulty:** Easy
- **Date completed:** 1st of August 2026

## Summary

Day 06 of Hacker Holidays was a pure OSINT room, no exploitation involved - just reading a leaked Discord conversation, tracking down a target's Gravatar profile by email, and decoding a Base64 string to get the flag.

## Walkthrough

### Step 1: Reading the attached conversation

The room provided an attached `.png` file showing a Discord conversation between two users, **Lambo** and **Ponzi**. Reading through it for clues was the starting point for the rest of the room.

### Step 2: Spotting the profile aggregator - starts with a "G"

In the conversation, Lambo mentions he rarely uses social media anymore, but that he has an account on a site where you can link all your other social media accounts together - and that the name starts with a **G**. After some digging, that tool turned out to be **Gravatar**, a free service that ties a public profile (photo, bio, linked social accounts) to your email address.

### Step 3: Grabbing the email from the conversation

Further down in the chat, Lambo gives Ponzi his email address so he can be contacted: `lambobytelotushotel@gmail.com`. Handy, since Gravatar lets you search for a profile directly by email - which returns that profile's URL.

### Step 4: Finding the string and decoding it

Opening the Gravatar profile URL turned up a string of letters and numbers sitting on the page. It turned out to be **Base64**, so it went into **CyberChef** for decoding - which produced the flag.

<img width="1533" height="555" alt="image" src="https://github.com/user-attachments/assets/547150dc-35ee-468b-a572-2572f67893d7" />


And voilà, there you have it!

## Flag
![redacted](https://img.shields.io/badge/-REDACTED-000000) - to avoid spoilers, correct flag will be posted after the event is concluded.

## Tools Used
- Google
- Gravatar
- CyberChef

## Lessons Learned

- Casual conversations (even screenshots) can leak more than people realize - an offhand mention of "an old profile site" and an email address were enough to crack this room.
- Gravatar is a great reminder that an email address alone can lead straight to a public profile - searching by email returns the profile URL directly.
- CyberChef is a solid go-to for quickly identifying and decoding encoded strings like Base64.
