# Day 02
- **Challenge name:** [Room 404]([https://tryhackme.com/room/hh-theconciergeknows-2d7eb4d9](https://tryhackme.com/room/hh-room404-804573bf))
- **Category:** Web Exploitation / Source Code Exposure
- **Difficulty:** Easy
- **Date completed:** 28th of July 2026

## Summary
This challenge picks up where Day 01 left off - the Byte Lotus's guest-experience platform went live in a hurry, and the night-shift developer shipped more than the website. The task is simple on paper: dump the exposed source code and find the flag. The real lesson is in *how* something as small as a stray `.git` folder can hand over a full history of a codebase - including things the developer thought they'd already deleted.

## Exploitation / Walkthrough
### Step 1
Before diving into any dumping tools, I did a quick sanity check to see if `.git` was actually exposed on the web server:

```bash
curl -i http://10.114.133.85:8080/.git/HEAD
```

This returned:
```
ref: refs/heads/main
```

`.git/HEAD` is a tiny, harmless-looking file that every git repo has, so it's a perfect canary - if the server returns its contents instead of a 404, that confirms a full `.git` directory is sitting there, publicly reachable.

### Step 2
With the exposure confirmed, I reconstructed the entire repository using `git-dumper`, which crawls the git objects/refs/packs (since there's no directory listing) and rebuilds a working repo locally:

```bash
pip install git-dumper --break-system-packages
git-dumper http://10.114.133.85:8080/.git/ ./bytelotus
```

This gave me a normal, fully-functional local git repository - as if I'd cloned it directly.

### Step 3
Instead of only checking the current state of the files (`HEAD`), I walked the entire commit history across all branches, including the diffs for every commit:

```bash
cd bytelotus
git log --all -p | less
```

- `--all` ensures every branch/ref is checked, not just the current one - important in case something was committed on a branch that was never merged into `main`.
- `-p` shows the actual diff content of every commit, not just commit messages.
- Piping into `less` let me scroll through the full history manually, and the flag turned up in the diff of an old commit.

This pulled the flag straight out of an old commit that had since been "cleaned up" on `HEAD`.

And voilà, there you have it!

## Flag
![redacted](https://img.shields.io/badge/-REDACTED-000000) - to avoid spoilers, correct flag will be posted after the event is concluded.

## Lessons Learned
Key takeaway: deploying a project folder as-is (instead of just the built/exported app) can silently ship the entire `.git` directory to production, and with it, the *complete* history of the codebase - not just its current state. A developer removing a secret in a later commit gives a false sense of security; the secret is still sitting in the object database unless history is deliberately rewritten.
