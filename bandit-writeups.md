# OverTheWire: Bandit — Write-ups

Self-directed, ungraded security practice, separate from formal coursework — working toward the CEH/OSCP path. [Bandit](https://overthewire.org/wargames/bandit/) is a wargame of 34 levels (0–33), each teaching a specific Linux/security concept, where you use what you learn on one level to find the password for the next.

**No passwords or private keys are recorded here, on purpose.** Bandit passwords prove you solved each level yourself — publishing them would let anyone skip straight to the next level, and undermines the actual point of doing this. What's documented instead is the *technique*: what the level was testing, what didn't work, and why the real fix worked. That's more useful to show anyway — it's proof of understanding, not proof of access. Bandit itself issues no certificate; the value is being able to actually talk through how each of these was solved.

Currently around level 18 of 34.

## Format

Each entry: what the level required, the wrong turns taken (if any), the concept that actually solved it, and what it taught. No commands with real output, no credentials, no private keys.

## Levels 0–4 — [to fill in]

*(Covered basic SSH login, navigating hidden/dash-prefixed/space-containing filenames, and using `file` to identify a file's actual type before trying to read it. Write up properly once revisited — the goal is to explain the concept, not just log that the level was passed.)*

## Level 5 — `find` across a directory maze

**What it required:** A folder tree of nested subdirectories full of decoy files, with one real target file identifiable only by its exact properties (a specific size, in bytes).

**What actually solved it:** `find` with `-type f` and `-size` flags to search the filesystem by file properties rather than by opening and reading every file by hand.

**What it taught:** `find` operates on filesystem metadata — name, size, owner, type, location — never on what's actually written inside a file. That distinction (metadata search vs. content search) is what separates `find` from `grep`, and got confused with it later on — see Level 7 below.

## Level 6 — `find` system-wide by owner and group

**What it required:** Locating one specific file somewhere on the entire server (not just a local folder), identifiable by owner, group, and exact size.

**What actually solved it:** `find` again, this time with `-user`, `-group`, and `-size` flags searching from the filesystem root, redirecting permission-denied errors elsewhere so they didn't drown the real result.

**What it taught:** The same `find`-searches-properties principle as Level 5, scaled up — and that a real system has thousands of files you don't have permission to even see, so filtering error noise (`2>/dev/null`) is part of using `find` productizedly, not a separate trick.

## Level 7 — `grep` for a known word

**What it required:** A large text file, with the password sitting on the one line next to a specific known word ("millionth").

**What actually solved it:** `grep` for that exact word, printing just the matching line instead of scrolling through the whole file.

**What it taught:** This is where `find` vs `grep` actually got sorted out for good: `find` answers "where is a file with these properties?" — `grep` answers "which line in a file I already have contains this text?" They were being treated as interchangeable before this, since both take a search term as an argument, but they operate on completely different things (filesystem metadata vs. file content).

## Level 9 — `strings` on a binary-garbage file, then re-reading own output

**What it required:** A file that `cat` printed as unreadable, corrupted-looking symbols — the password was a human-readable string buried inside otherwise binary data, marked by a line of `=` characters just before it.

**What tripped me up:** `strings` filtered the binary noise down to readable text, but the actual output was still ~150 lines of scattered fragments, and the first pass through it missed the real line entirely.

**What actually solved it:** Going back over the same `strings` output a second time, more carefully, and spotting the one line following a `====` marker that the first read-through had scanned past.

**What it taught:** Two things. First, mechanically: `cat` prints a file's raw bytes regardless of whether they're printable text or not, while `strings` filters a file down to only the sequences of bytes that fall in the printable-ASCII range — which is why the same file looks like garbage under `cat` and like (mostly) real words under `strings`. Second, less technical but arguably more important: sometimes the answer isn't hidden by a clever trick, it's just sitting in a wall of noise, and re-reading your own already-correct output more carefully solves it faster than trying a different command.

## Level 12 — nested compression and `xxd`

**What it required:** A file that turned out to be several layers of compression/encoding stacked on top of each other (hexdump encoding plus multiple rounds of archive formats), needing `file` to identify what each layer actually was before knowing which tool undid it.

**What actually solved it:** Running `file` on the current version of the data at every step to find out what it actually was (rather than guessing), then peeling off one layer at a time — `xxd -r` to reverse a hexdump back to binary, then whichever decompression tool `file` said was needed at each stage (gzip, bzip2, tar, etc. in some order).

**What it taught:** Don't assume you know what a file is from its extension or from what the level says at a glance — `file` exists specifically to tell you the real format, and with layered/obfuscated data, checking after every single transformation is the only reliable way through, not guessing the whole chain up front.

## Level 13 — private key transfer off the game server

**What it required:** An SSH private key that needed to be copied from the Bandit server down to the local machine before it could be used to authenticate as the next level's user (this is the level where you leave the game server for the first time — see Level 14's entry in a moment for what came after).

**What tripped me up:** `scp` syntax on Windows, specifically the difference between `ssh` (which just opens a connection, no source/destination distinction) and `scp` (which needs `user@host:/path` on one side to know which argument is the remote file). Also mixed up `-p` and `-P`: lowercase `-p` on `scp` means "preserve file timestamps/permissions," not port — capital `-P` is the port flag, unlike `ssh` which uses lowercase `-p` for port. Same letter, two different tools, two different meanings.

**What actually solved it:** `scp -P <port> user@host:/absolute/remote/path .` — the `user@host:` prefix with a colon is what tells `scp` "this side is remote," and the trailing `.` means "save it in my current folder."

**What it taught:** Flags aren't universal across tools just because they look similar — `-p` means something completely different in `scp` than it does in `ssh`, purely because that letter was already taken for something else in `scp`'s own design. Worth checking `--help`/`man` per-tool rather than assuming a flag transfers over.

## Level 14 — using the transferred key, then reading the password

See below — this expands the entry that was already here.

**What it required:** Using the private key retrieved in Level 13 to authenticate via SSH, then reading the next password from an absolute system path (`/etc/bandit_pass/bandit14`) rather than anywhere inside the home directory.

**What tripped me up:** After transferring the key to Windows, SSH refused to use it — Windows file permissions (ACLs) on the key file were too permissive, and SSH treats an overly-open private key file as a security risk and won't touch it until that's fixed.

**What actually solved it:** Fixing the transferred key file's Windows permissions so only the owner could read it, which let SSH accept it as valid. Then simply `cat`-ing the absolute path given in the level's own instructions — the password isn't discoverable by browsing (`ls` in the home directory shows nothing pointing to it), because it lives in a completely different, system-owned part of the filesystem. Some Bandit levels are "search the filesystem yourself" (Level 5, Level 6); this one is "read the instructions carefully," a different skill entirely.

**What it taught:** SSH checks that a private key file is only readable by its owner as a defense against a key sitting somewhere any other user or process could read it — this is a real security control (the same principle behind `~/.ssh` and its keys being `700`/`600` by default on Linux), not an arbitrary SSH quirk. Also: not every level rewards exploring — knowing when to re-read the brief instead of searching harder is its own skill.

## Levels 15–18 — [to fill in]

*(In progress — this range moves into `openssl`/SSL-wrapped connections and networking/port-scanning territory (`nmap`), a different skill set from the filesystem-focused early levels. Write up properly once each is solved with enough understanding to explain without notes.)*

## What's next

Plan (as of finishing roughly level 15, later revised) is: aim to get well through the 20s, since that covers Linux fundamentals, permissions, basic scripting, and `nmap` — the broadly useful stuff for a security/network engineering direction rather than the pentest-specific binary exploitation and reverse-engineering territory further in. TryHackMe's free tier starts once Bandit stops teaching new things rather than waiting for all 34 levels, then Hack The Box after that.
