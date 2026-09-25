# OverTheWire: Bandit — Write-ups

Self-directed, ungraded security practice, separate from formal coursework — working toward the CEH/OSCP path. [Bandit](https://overthewire.org/wargames/bandit/) is a wargame of ~34 levels, each teaching a specific Linux/security concept, where you use what you learn on one level to find the password for the next.

**No passwords or private keys are recorded here, on purpose.** Bandit passwords prove you solved each level yourself — publishing them would let anyone skip straight to the next level, and undermines the actual point of doing this. What's documented instead is the *technique*: what the level was testing, what didn't work, and why the real fix worked. That's more useful to show anyway — it's proof of understanding, not proof of access.

Currently around level 18 of 34.

## Format

Each entry: what the level required, the wrong turns taken (if any), the concept that actually solved it, and what it taught. No commands with real output, no credentials, no private keys.

## Level 14 — SSH key across hosts, permission fixes

**What it required:** Using a password from one level to authenticate to a Bandit level's own SSH port, then retrieving an SSH private key that had to be used to connect to a *different* host (`localhost`, but on the target's own address, not straightforward to just "run from where I already was").

**What tripped me up:** Getting the private key file itself off the target and usable locally — `scp` to pull the key file across, then hitting SSH's file-permission requirements on the key once transferring it to a Windows machine (SSH refuses to use a private key file with overly permissive Windows ACLs).

**What actually fixed it:** Fixing the transferred key file's permissions so SSH would accept it as a valid private key, rather than rejecting it outright for being "too open."

**What it taught:** SSH doesn't just check that a key is *correct* — it checks that the key file itself is only readable by its owner, as a defense against a key sitting somewhere any other user or process could read it. This is a File permissions security control, not just an SSH quirk, and it's the same principle behind why `~/.ssh` and its keys are `700`/`600` by default on Linux.

## Levels 0–13 — [to fill in]

*(Earlier levels covered: basic SSH login, `ls`/`cat`/file navigation, hidden files, file permissions and `find`, string/binary manipulation (`strings`, `base64`, compressed/archived files), basic process/port inspection, and simple cron/setuid concepts. Write these up properly once revisited — the goal is to explain the concept, not just log that the level was passed.)*

## Levels 15–18 — [to fill in]

*(In progress. Update as each is actually completed and understood well enough to explain without notes.)*

## What's next

Plan is to try TryHackMe's free tier once further into Bandit — not waiting for full completion — then move on to Hack The Box's beginner tracks.
