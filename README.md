# Lab 27: Safely Editing System Databases — `vipw`, `vigr`, `visudo`

**Series:** File Operations & Shell Fundamentals · **Lab 27 of the Novice → RHCA path**  
**Certifications covered:** RHCSA EX200 (user/group management, sudoers), RHCE EX294 (Ansible `user`/`group`/`lineinfile` on these files), CKA (RBAC binding files use the same care), RHCA building blocks (RH342 emergency password edits, RH358 service-account hygiene, RH236 brick-mount user mapping)  
**Prerequisite:** Labs 05–26 (filesystem, `cat`, `grep`, `sed`, `awk`, `vi`)  
**Time Estimate:** 45–60 minutes  
**Difficulty arc:** Tasks 1–6 foundation · 7–13 practical · 14–18 advanced · 19–20 exam-realistic

---

## 🎯 Objective

By the end of this lab you will edit `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow`, and `/etc/sudoers` **safely** — meaning **with locking, syntax checking, and a known-good restore path**. You will never again break `sudo`, lock yourself out, or cause `getent` to return empty results because of a missing colon. You will know the difference between `vipw`, `vipw -s`, `vigr`, `vigr -s`, and `visudo`, and when each one is the correct answer.

---

## 🧠 Concept: These Files Are a Database

`/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow`, and `/etc/sudoers` look like text files — but they are read **constantly** by every process that authenticates or checks identity. A single broken line during a write can:

- Lock all users out (corrupt `/etc/shadow`).
- Break `sudo` permanently (corrupt `/etc/sudoers`).
- Cause `id`, `ls -l`, `chown` to misbehave (corrupt `/etc/passwd` or `/etc/group`).

The "safe editing" family of tools (`vipw`, `vigr`, `visudo`) gives you three guarantees:

```
1. LOCK    — prevent two admins (or a script) from writing simultaneously
2. EDIT    — open in $EDITOR with the right backup/temp-file workflow
3. VERIFY  — refuse to save the file if syntax/integrity is broken
```

### The five files and their wrappers

| File | Format | Safe editor | Notes |
|---|---|---|---|
| `/etc/passwd` | 7 colon-separated fields | `vipw` | Public — readable by all |
| `/etc/shadow` | 9 colon-separated fields | `vipw -s` | Sensitive — root-only |
| `/etc/group` | 4 colon-separated fields | `vigr` | Public |
| `/etc/gshadow` | 4 colon-separated fields | `vigr -s` | Sensitive — root-only |
| `/etc/sudoers` (+ `/etc/sudoers.d/*`) | Custom syntax | `visudo` | NEVER edit directly |

> **Senior-engineer rule:** Use `vipw` / `vigr` / `visudo` **always** — even for "just one character" changes. The locking and the syntax check are insurance you can't buy after you broke `sudo`.

### Field layouts you'll memorize

`/etc/passwd`:

```
username : x : UID : GID : GECOS : home : shell
   1       2   3    4     5       6      7
```

`/etc/shadow`:

```
username : password : lastchange : min : max : warn : inactive : expire : reserved
   1         2          3          4     5     6      7           8        9
```

`/etc/group`:

```
groupname : x : GID : member1,member2,...
   1         2  3     4
```

`/etc/gshadow`:

```
groupname : password : admins : members
   1          2          3       4
```

---

## 📚 Command Reference

### `vipw` and `vigr`

| Command | Purpose |
|---|---|
| `sudo vipw` | Edit `/etc/passwd` with lock + integrity check |
| `sudo vipw -s` | Edit `/etc/shadow` |
| `sudo vigr` | Edit `/etc/group` |
| `sudo vigr -s` | Edit `/etc/gshadow` |
| `sudo vipw -q` | Quiet — useful in scripts |

These tools:

1. Create a lock file (`/etc/passwd.lock`, etc.).
2. Open a working copy in your `$EDITOR` (`vi` by default).
3. After save, verify the file's structure (`pwck -r` / `grpck -r`).
4. Atomically replace the original.
5. Release the lock.
6. Some implementations also offer to also update the matching shadow file (you'll be prompted).

### `visudo`

| Command | Purpose |
|---|---|
| `sudo visudo` | Edit `/etc/sudoers` safely |
| `sudo visudo -f /etc/sudoers.d/NAME` | Edit a drop-in fragment |
| `sudo visudo -c` | Check syntax only |
| `sudo visudo -c -f /etc/sudoers.d/NAME` | Check a fragment |
| `sudo EDITOR=vim visudo` | Use vim instead of default editor |

`visudo` runs `sudoers` syntax validation on save and refuses to write if invalid.

### Companion verification tools

| Tool | Purpose |
|---|---|
| `pwck -r` | Read-only check of `/etc/passwd` / `/etc/shadow` |
| `grpck -r` | Read-only check of `/etc/group` / `/etc/gshadow` |
| `getent passwd USER` | NSS-aware lookup of a user (covers LDAP/AD too) |
| `getent group GROUP` | NSS-aware group lookup |
| `id USER` | UID, GID, supplementary groups |
| `chage -l USER` | Show shadow aging settings |
| `passwd -S USER` | Single-line account status (`L`/`P`/`NP`/`LK`) |

---

## 🛣️ RHCA Pathway Sidebar

| Cert level | Why this lab matters |
|---|---|
| **Foundation** | The day you break `/etc/passwd` is the day you wish you had used `vipw` |
| **RHCSA EX200** | User/group management appears in nearly every exam; `visudo` is mandatory |
| **RHCE EX294** | Ansible's `user`/`group` modules manipulate the same files — but for **manual rescue** you need `vipw` |
| **CKA** | RBAC binding YAML is its own world, but Linux nodes still rely on these files for kubelet user contexts |
| **RHCA — RH342 (Troubleshooting)** | Emergency `vipw` rescue when shadow gets corrupted by a bad script |
| **RHCA — RH358 (Services)** | Service-account drift — `vigr -s` to audit group memberships |
| **RHCA — RH236 (Storage)** | NFS / GlusterFS UID mapping reconciliation |

---

## 🔧 The 20 Tasks

> Each task ends with three short callouts: **Commands**, **Output decoded**, and **Troubleshoot**.

> ⚠️ **Most of this lab modifies real authentication files. Use a VM or a disposable cloud host whenever possible. Always make backups. Have a second sudo session open in case you lock yourself out.**

---

### Task 1 — Inspect `/etc/passwd` and confirm structure

**Purpose:** Read the database first — never edit what you don't understand.

```bash
head -3 /etc/passwd
awk -F: '{ print NF, $1 }' /etc/passwd | sort -u | head -5
```

**Expected output (excerpt):**

```
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
7 root
7 bin
7 daemon
7 adm
7 lp
```

**Human-Readable Breakdown**

> "Hey bash, show me the first three lines of `/etc/passwd` so I can see what a record looks like. Then run `awk` to count how many colon-separated fields are on every line — every record should have exactly 7. If any line shows a different number, the file is corrupt."

**Reading it left to right (line 1):**
- `head -3` → "show me the first **3** lines of the file."
- `/etc/passwd` → "the user database."

**Reading it left to right (line 2):**
- `awk` → "a text-processing tool that splits each line into fields."
- `-F:` → "**F**ield separator = `:` (colon). Without this, `awk` would split on spaces."
- `'{ print NF, $1 }'` → "for each line, print `NF` (the **N**umber of **F**ields) followed by `$1` (the first field — the username)."
- `/etc/passwd` → "the file to process."
- `|` → "pipe — feed `awk`'s output into the next command."
- `sort -u` → "**sort** the lines, `-u` = remove duplicates."
- `|` → "pipe again."
- `head -5` → "show only the first 5 results."

**The story:** Before editing a database, you **read** it. `/etc/passwd` is colon-delimited with exactly 7 fields per record: `username:x:UID:GID:GECOS:home:shell`. The `awk` one-liner verifies that invariant: every line should report `7` as its field count. If any line shows `6` or `8`, you have corruption — likely a missing or extra colon — and `pwck` will help locate it.

**Commands**

| Token | Meaning |
|---|---|
| `head -3 /etc/passwd` | First three records |
| `awk -F: '{print NF, $1}'` | Field count plus username |

**Output decoded**

| Field count | Meaning |
|---|---|
| `7` | Correct — every record has exactly 7 fields |
| Any other number | Corruption — investigate immediately |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Some lines show ≠ 7 | Run `sudo pwck -r` — it will identify the bad lines |

---

### Task 2 — Inspect `/etc/shadow` (root only)

**Purpose:** Confirm you can read shadow as root, and understand the format.

```bash
sudo head -3 /etc/shadow
sudo awk -F: '{ print NF, $1 }' /etc/shadow | sort -u
```

**Expected output (excerpt):**

```
root:$6$xxx$yyy:19500:0:99999:7:::
bin:*:18923:0:99999:7:::
daemon:*:18923:0:99999:7:::
9 root
9 bin
9 daemon
…
```

**Human-Readable Breakdown**

> "Same thing as Task 1, but for `/etc/shadow` — the sensitive password-hash file. It needs `sudo` because only root can read it. Verify each record has exactly 9 fields."

**Reading it left to right (line 1):**
- `sudo` → "**s**uperuser **do** — run the next command as root."
- `head -3 /etc/shadow` → "show me the first 3 records of the shadow file."

**Reading it left to right (line 2):**
- `sudo awk -F: '{ print NF, $1 }' /etc/shadow` → "as root, field-split on `:`, print field count + username."
- `| sort -u` → "sort, dedupe."

**The story:** `/etc/shadow` is the secret sibling of `/etc/passwd`. While `passwd` is world-readable (mode 644), `shadow` is root-only (mode 0000 or 0640) because it holds the actual password hashes. Shadow has **9 fields**: `username:hash:lastchange:min:max:warn:inactive:expire:reserved`. The hash format prefix tells you the algorithm: `$6$` = SHA-512, `$y$` = yescrypt (RHEL 9+), `*` or `!` = the account has no password / is locked. If a field count comes back as anything other than 9, the file is broken — use `pwck -r` to locate the bad line.

**Commands**

| Token | Meaning |
|---|---|
| `sudo head -3 /etc/shadow` | First three records |
| Field count | Should be 9 |

**Output decoded**

| Field | Meaning |
|---|---|
| 2 | Hashed password (`$6$...` = SHA-512; `*` or `!` = locked) |
| 3 | Last password change (days since 1970-01-01) |
| 4 | Min days before next change |
| 5 | Max days valid |
| 6 | Warn days |
| 7 | Inactive days after expiry |
| 8 | Account expire date |

**Why a sysadmin needs this on RHCSA EX200:** Password aging tasks reference these fields directly.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Permission denied` | Use `sudo` |
| `$y$...` instead of `$6$...` | yescrypt (RHEL 9+) — also fine |

---

### Task 3 — First safe edit: `vipw` on `/etc/passwd`

**Purpose:** Open the file the right way. **Don't change anything yet** — just learn the workflow.

```bash
sudo vipw
```

You'll see a status message like:

```
You are using shadow passwords on this system.
Would you like to edit /etc/shadow now [y/n]?
```

Press `n` (we'll come back to shadow in Task 4). The editor opens the working copy. **Do nothing destructive**:

```
:q!     (quit without saving — buffer discarded, lock released)
```

**Human-Readable Breakdown**

> "Open `/etc/passwd` the **safe** way — using `vipw` instead of plain `vi`. `vipw` will lock the file, give you a working copy in the editor, and run an integrity check on save. Don't actually change anything yet; this run is just to learn the workflow."

**Reading it left to right:**
- `sudo` → "run as root (only root can edit identity files)."
- `vipw` → "**vi** for **p**ass**w**d — wrapper around your editor that adds locking + integrity check."
- The prompt **"Would you like to edit /etc/shadow now [y/n]?"** → "some distros chain the two edits. Answer `n` for now."
- `:q!` → "in vi: **q**uit, `!` = force / discard changes. This releases the lock without modifying the file."

**The story:** `vipw` is the production-grade way to open `/etc/passwd`. It does three things plain `vi` doesn't: (1) creates `/etc/passwd.lock` so two admins can't write simultaneously, (2) opens a **working copy** in `$EDITOR` (defaults to `vi`), and (3) runs `pwck` after you save to refuse the write if the file is structurally broken. The `:q!` exit is the safe "I changed my mind" — buffer discarded, lock released, original untouched. Practicing this dry run is **the** muscle memory that prevents disasters when you're tired and need to fix a real corruption at 3am.

**Commands**

| Token | Meaning |
|---|---|
| `sudo vipw` | Lock + edit `/etc/passwd` |
| `:q!` | Quit without saving |

**Output decoded**

| Element | Meaning |
|---|---|
| The "edit /etc/shadow now?" prompt | Some distros chain `passwd` + `shadow` edits |
| Lock created during edit | Other processes that try `vipw` will wait or fail |
| `:q!` returns | Lock released, file unchanged |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `vipw: cannot lock /etc/passwd; try again later` | Another `vipw` is running — find with `ls /etc/passwd.lock` and `ps -ef \| grep vipw` |
| Wrong editor opens | `sudo EDITOR=vim vipw` or set `export EDITOR=vim` first |

---

### Task 4 — Add a real user via the right tool, then audit with `vipw`

**Purpose:** Do **not** hand-add users to `/etc/passwd`. Use `useradd` — then practice safe inspection with `vipw`.

```bash
sudo useradd -m -s /bin/bash -c "Lab27 Tester" labtester
sudo passwd labtester      # set a temporary password
id labtester
getent passwd labtester
```

**Expected output:**

```
uid=1002(labtester) gid=1002(labtester) groups=1002(labtester)
labtester:x:1002:1002:Lab27 Tester:/home/labtester:/bin/bash
```

Now audit:

```bash
sudo vipw
```

In `vi`, search:

```
/^labtester:
:q       (quit — no changes)
```

**Human-Readable Breakdown**

> "Don't hand-edit `/etc/passwd` to create a user — that's the recipe for disaster. Use `useradd` (with `-m` to create the home directory, `-s` to set the shell, `-c` to set the full name). Set a password. Then verify with `id` and `getent`. Finally, open `/etc/passwd` safely with `vipw` to see the new line — but don't change anything."

**Reading it left to right (line 1):**
- `sudo useradd` → "as root, **add** a **user**."
- `-m` → "create their home directory (**m**ake home)."
- `-s /bin/bash` → "**s**et their default shell."
- `-c "Lab27 Tester"` → "set the **c**omment field (a.k.a. GECOS, usually the full name)."
- `labtester` → "the username."

**Reading it left to right (lines 2-4):**
- `sudo passwd labtester` → "set the password interactively. You'll be prompted twice."
- `id labtester` → "show UID, GID, and supplementary group memberships."
- `getent passwd labtester` → "**get ent**ry from NSS (Name Service Switch). Works even if the user lives in LDAP/AD, not just `/etc/passwd`."

**Reading it left to right (the vi search):**
- `sudo vipw` → "open `/etc/passwd` safely."
- `/^labtester:` → "in vi, `/` = search forward. The pattern `^labtester:` means *line starting with `labtester:`*. Jumps to the new entry."
- `:q` → "quit (no `!` needed because nothing changed)."

**The story:** This is the **correct** order of operations: `useradd` creates the user *properly* — `/etc/passwd` entry, `/etc/shadow` entry, home directory, ownership, default dotfiles all in one atomic action. Then `passwd` sets the credential. Then you verify three different ways: `id` (low-level kernel view), `getent` (NSS-aware lookup that works with LDAP/AD), and `vipw` (visual confirmation in the file itself). On the RHCSA, this is the exam-realistic pattern.

**Commands**

| Token | Meaning |
|---|---|
| `useradd -m -s SHELL -c COMMENT NAME` | Properly add the user (home dir, shell, GECOS) |
| `passwd NAME` | Set the password |
| `id NAME` | Verify UID/GID/groups |
| `getent passwd NAME` | NSS-aware lookup |
| `vipw` + search | Confirm the entry visually |

**Output decoded**

| Element | Meaning |
|---|---|
| Tester entry visible at end of `/etc/passwd` | Created by `useradd` |
| `home/labtester` exists | Because of `-m` |

**Why a sysadmin needs this on RHCSA EX200:** Demonstrate the correct workflow — `useradd` for creation, `vipw` for inspection / careful edits.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `useradd` already taken | Pick a different name; or use `id labtester` to confirm |
| Want to delete later | `sudo userdel -r labtester` (cleanup at end of lab) |

---

### Task 5 — Change the user's GECOS (full name) via `vipw`

**Purpose:** Practice a **safe** edit you can verify with `getent`. `chfn` exists but `vipw` is exam-realistic.

```bash
sudo vipw
```

Inside:

```
/^labtester:
A             (cursor to end-of-line)
Esc
:s|labtester:x:\([0-9]\+\):\([0-9]\+\):.*:|labtester:x:\1:\2:Updated GECOS Field:|
:wq           (save — vipw runs the verifier)
```

After save, verify:

```bash
getent passwd labtester
```

**Expected output:**

```
labtester:x:1002:1002:Updated GECOS Field:/home/labtester:/bin/bash
```

**Human-Readable Breakdown**

> "Open `/etc/passwd` safely. Find the `labtester` line. Replace the GECOS (full-name) field with new text using a vi substitute command — but preserve the UID and GID with back-references. Save. `vipw` will run the integrity check before writing. Verify with `getent`."

**Reading it left to right (the vi commands):**
- `sudo vipw` → "open `/etc/passwd` safely."
- `/^labtester:` → "search for the line starting with `labtester:`."
- `A` → "in vi: jump to **end of line** and switch to insert mode (capital A)."
- `Esc` → "leave insert mode."
- `:s|labtester:x:\([0-9]\+\):\([0-9]\+\):.*:|labtester:x:\1:\2:Updated GECOS Field:|` → "**s**ubstitute on the current line."
  - `|` → "delimiter (used instead of `/` because the path fields contain slashes)."
  - `labtester:x:\([0-9]\+\):\([0-9]\+\):.*:` → "match pattern: `labtester:x:`, then capture group 1 = digits (UID), then `:`, then capture group 2 = digits (GID), then `:`, then anything up to the next `:`."
  - `\([0-9]\+\)` → "in basic regex, `\(...\)` = capture group; `[0-9]\+` = one or more digits."
  - `labtester:x:\1:\2:Updated GECOS Field:` → "replacement: keep `\1` (the captured UID) and `\2` (the captured GID), but replace the GECOS field with the new text."
- `:wq` → "**w**rite and **q**uit. `vipw` runs `pwck` before writing — if anything's broken, it refuses to save."

**Reading the verify step:**
- `getent passwd labtester` → "lookup labtester via NSS to confirm the GECOS change took effect."

**The story:** This task practices a **safe** structured edit with full back-reference preservation. In production, you'd use `chfn -f "Updated GECOS Field" labtester` — but on the RHCSA exam, the grader might want to see that you can do it manually. The back-reference trick (`\1`, `\2`) is gold: it lets you change one field without accidentally retyping (and possibly mangling) the UID or GID. The `vipw`-wrapped save is the safety net — if your regex blew up and produced an invalid line, the integrity check catches it before the file hits disk.

**Commands**

| Token | Meaning |
|---|---|
| `:s\|A\|B\|` | Substitute on current line; `\|` delimiter so `/` paths don't conflict |
| `\1`, `\2` | Back-references for UID/GID (preserved) |
| `:wq` | Save and trigger `vipw`'s integrity check |

**Output decoded**

| Element | Meaning |
|---|---|
| `Updated GECOS Field` | Field 5 changed; the rest preserved |
| No verifier complaint | File is structurally valid |

**Why a sysadmin needs this on RHCSA EX200:** Hands-on familiarity with editing the database — even though `chfn -f "New Name" labtester` is faster.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `vipw` rejects the file with a `pwck` error | Pick a clean re-edit; press `e` (edit again) at the prompt |
| Changes vanished | You used `:q!` — repeat with `:wq` |

---

### Task 6 — Edit `/etc/shadow` to set aging policy via `vipw -s`

**Purpose:** Use `chage` first (the right way), but understand the shadow fields and how `vipw -s` lets you fix damage.

```bash
sudo chage -l labtester
sudo chage -m 1 -M 60 -W 7 labtester
sudo chage -l labtester
sudo vipw -s
```

Inside the editor:

```
/^labtester:
:wq       (don't change anything — just confirm the integrity hook fires)
```

**Expected output (`chage -l` after):**

```
Last password change                                : May 21, 2026
Password expires                                    : Jul 20, 2026
Password inactive                                   : never
Account expires                                     : never
Minimum number of days between password change      : 1
Maximum number of days between password change      : 60
Number of days of warning before password expires   : 7
```

**Human-Readable Breakdown**

> "First show me the current password-aging policy for labtester. Then set new aging rules: minimum 1 day between password changes, maximum 60 days before forced change, warn 7 days before expiry. Show the aging again to confirm. Finally, open `/etc/shadow` safely with `vipw -s` to see the underlying fields — just to confirm the integrity hook fires when you save."

**Reading it left to right (line 1):**
- `sudo chage -l labtester` → "**ch**ange **age** policy, `-l` = **list**. Show current aging settings for labtester."

**Reading it left to right (line 2):**
- `sudo chage -m 1 -M 60 -W 7 labtester` → "set aging."
- `-m 1` → "**m**inimum days between password changes = 1."
- `-M 60` → "**M**aximum days the password is valid = 60."
- `-W 7` → "**W**arning days before expiry = 7."
- `labtester` → "target user."

**Reading it left to right (line 3):**
- `sudo chage -l labtester` → "re-list to confirm the changes took effect."

**Reading it left to right (line 4 and vi):**
- `sudo vipw -s` → "**vi p**ass**w**d, `-s` = **s**hadow. Open `/etc/shadow` safely."
- `/^labtester:` → "find labtester's shadow line."
- `:wq` → "save without changes — just to verify `vipw -s` runs the integrity hook on every save."

**The story:** `chage` is the right tool for password aging — it edits `/etc/shadow` fields 3-8 (lastchange, min, max, warn, inactive, expire) for you. `vipw -s` is the rescue tool when `chage` can't fix something (e.g., a corrupted hash field). On the RHCSA, password aging is a recurring objective: "Set user X's password to expire every 60 days with a 7-day warning." The fastest correct answer is `sudo chage -M 60 -W 7 X`, then verify with `chage -l X`.

**Commands**

| Token | Meaning |
|---|---|
| `chage -l USER` | List aging |
| `chage -m N` | Min days between changes |
| `chage -M N` | Max validity in days |
| `chage -W N` | Warning days |
| `vipw -s` | Open `/etc/shadow` safely |

**Output decoded**

| Field | Meaning |
|---|---|
| `Maximum…60` | Field 5 of shadow set to 60 |
| `Minimum…1` | Field 4 set to 1 |
| `Warning…7` | Field 6 set to 7 |

**Why a sysadmin needs this on RHCSA EX200:** Password aging is an exam objective. Verify both via `chage -l` and (when truly needed) `vipw -s`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `chage` denies | Need `sudo` or PAM constraints — check `man chage` |
| Want to lock | `sudo passwd -l labtester` adds `!` prefix to hash |

---

### Task 7 — Add a group with `groupadd`, then verify with `vigr`

**Purpose:** Group creation through the right tool; safe inspection with `vigr`.

```bash
sudo groupadd developers
sudo gpasswd -a labtester developers
getent group developers
id labtester
sudo vigr
```

Inside:

```
/^developers:
:q
```

**Expected output (`getent`):**

```
developers:x:1500:labtester
```

**Human-Readable Breakdown**

> "Create a group called `developers` using the proper tool. Add `labtester` as a member of that group. Verify with `getent` and `id`. Then open `/etc/group` safely with `vigr` and look at the new line — don't change anything."

**Reading it left to right:**
- `sudo groupadd developers` → "**add** a new **group** called `developers`."
- `sudo gpasswd -a labtester developers` → "**g**roup **passwd** utility, `-a` = **add** a user. Add `labtester` as a member of `developers`."
- `getent group developers` → "**get ent**ry from NSS for the `developers` group. Confirms the group exists and shows its members."
- `id labtester` → "shows labtester's UID, primary GID, and supplementary group memberships — `developers` should now appear."
- `sudo vigr` → "**vi g**roup — wrapper around your editor that adds locking + integrity check for `/etc/group`."
- `/^developers:` → "in vi, find the line starting with `developers:`."
- `:q` → "quit; nothing changed."

**The story:** Groups are colon-delimited like passwd, but only 4 fields: `groupname:x:GID:members`. `groupadd` creates the entry. `gpasswd -a` adds a member to the comma-separated member list at field 4. Why not edit the file directly with `vi`? Because if you make a typo (e.g., accidentally delete a `:`), the next program that calls `getgrnam()` gets garbage — and that includes the kernel during permission checks. `vigr` lets you inspect or recover the file safely with full locking and integrity verification.

**Commands**

| Token | Meaning |
|---|---|
| `groupadd NAME` | Create a group |
| `gpasswd -a USER GROUP` | Add USER as member of GROUP |
| `vigr` | Lock + open `/etc/group` |
| `getent group GROUP` | Verify |

**Output decoded**

| Field | Meaning |
|---|---|
| 1 | Group name |
| 2 | Password placeholder (`x` → see gshadow) |
| 3 | GID |
| 4 | Comma-separated member list |

**Why a sysadmin needs this on RHCSA EX200:** "Add user X to group Y" is a recurring objective.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Group exists | Drop the `groupadd` step or pick another name |

---

### Task 8 — Edit `/etc/group` via `vigr` to add a member manually

**Purpose:** Practice editing the member list directly. This **simulates** a recovery situation; in production, prefer `gpasswd`.

```bash
sudo useradd -m teststaff
sudo vigr
```

Inside `vi`:

```
/^developers:
A,teststaff     (append a comma + name to the end of line)
Esc
:wq
```

Verify:

```bash
getent group developers
id teststaff
```

**Expected output:**

```
developers:x:1500:labtester,teststaff
uid=1003(teststaff) gid=1003(teststaff) groups=1003(teststaff),1500(developers)
```

**Human-Readable Breakdown**

> "Create a second user `teststaff` with `useradd`. Then open `/etc/group` safely with `vigr`. Find the `developers` line. Jump to the end of it. Type `,teststaff` to append the new name to the comma-separated member list. Save. Verify with `getent` and `id`. (In production you'd use `gpasswd -a` — this task simulates a recovery scenario where you need to fix the file by hand.)"

**Reading it left to right:**
- `sudo useradd -m teststaff` → "create a second user with a home directory."
- `sudo vigr` → "open `/etc/group` safely."
- `/^developers:` → "in vi, search for the developers line."
- `A` → "in vi (capital A): jump to **end of line** and enter insert mode. (Lowercase `a` would only go after the character under the cursor.)"
- `,teststaff` → "type a comma and the new username, appending to the member list."
- `Esc` → "leave insert mode."
- `:wq` → "save. `vigr` runs `grpck` before writing."

**Reading the verify lines:**
- `getent group developers` → "show the group line — should now contain `labtester,teststaff`."
- `id teststaff` → "show teststaff's groups — should include `1500(developers)` as a supplementary group."

**The story:** This task demonstrates the **manual rescue** pattern — directly editing `/etc/group` to add a member. The `A`-then-type-comma-name trick is the cleanest vi keystroke combo for appending to a line. The `vigr` wrapper protects you: if you accidentally garble the line (e.g., delete a `:`), `grpck` rejects the save and the file stays intact. In production, prefer `gpasswd -a` for one-off changes — but knowing how to do this by hand is what saves your job during a 3am recovery.

**Commands**

| Token | Meaning |
|---|---|
| `useradd -m teststaff` | Make a second user |
| `vigr` | Safe edit `/etc/group` |
| `A` (in vi) | Append at end-of-line |
| `,teststaff` | Comma-separated new member |

**Output decoded**

| Element | Meaning |
|---|---|
| Member list now has both names | Edit succeeded |
| `id teststaff` reflects supplementary group | NSS picked it up |

**Why a sysadmin needs this on RHCA RH342:** Direct edits are sometimes the fastest rescue path.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `id` doesn't show the new group for an existing session | The user must log out / back in (or refresh `sg DEVELOPERS`) |

---

### Task 9 — Read `/etc/gshadow` via `vigr -s`

**Purpose:** Understand the lesser-known group shadow file.

```bash
sudo vigr -s
```

Inside:

```
/^developers:
:q
```

**Expected line (example):**

```
developers:!::labtester,teststaff
```

**Human-Readable Breakdown**

> "Open `/etc/gshadow` (the lesser-known group-shadow file) safely with `vigr -s`. Find the `developers` line. Quit without changes — this task is just to learn the file's structure."

**Reading it left to right:**
- `sudo vigr -s` → "**vi g**roup, `-s` = **s**hadow. Open `/etc/gshadow` safely."
- `/^developers:` → "search for the developers line."
- `:q` → "quit; nothing changed."

**Reading the line `developers:!::labtester,teststaff`:**
- Field 1 — `developers` → "group name."
- Field 2 — `!` → "group password (hash). `!` = locked (the usual safe state)."
- Field 3 — *empty* → "comma-separated list of **group administrators** (people allowed to use `gpasswd` on this group). Empty = no designated admins."
- Field 4 — `labtester,teststaff` → "comma-separated member list (mirrors `/etc/group` field 4)."

**The story:** Almost no one talks about `/etc/gshadow`. It exists for symmetry with `/etc/shadow`: it stores **group passwords** (rare — most groups never have one) and **group admins** (users allowed to manage membership without root). If field 2 is `!` or `*`, the group password is locked — that's the normal, safe state. If you ever see an actual hash there, someone enabled `gpasswd <group>` and set a password. On RHCA RH358, service-account hygiene includes verifying that no service group has a real password.

**Commands**

| Token | Meaning |
|---|---|
| `vigr -s` | Edit `/etc/gshadow` |
| `!` in field 2 | Group password locked (typical) |
| Field 3 | Group administrators (comma-separated) |
| Field 4 | Member list (mirrors `/etc/group` field 4) |

**Output decoded**

| Field | Meaning |
|---|---|
| `developers` | Group name |
| `!` | Locked password (good) |
| Empty admin field | No designated group admin |
| `labtester,teststaff` | Members |

**Why a sysadmin needs this on RHCA RH358:** Service accounts sometimes have group passwords misconfigured — `vigr -s` is the fix.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `/etc/gshadow` missing | Rare; recreate with `grpconv` from `/etc/group` |

---

### Task 10 — Validate the entire user/group database with `pwck` and `grpck`

**Purpose:** The hidden verification — `vipw`/`vigr` call these under the hood, but you can call them on demand.

```bash
sudo pwck -r
sudo grpck -r
echo "exit codes: pwck=$? grpck=$?"
```

**Expected output (healthy):**

```
user 'tss': directory '/dev/null' does not exist
user 'tss': program '/sbin/nologin' does not exist
no matching password file entry in /etc/shadow
…
exit codes: pwck=0 grpck=0
```

**Human-Readable Breakdown**

> "Run the read-only integrity checks against the user database (`pwck`) and the group database (`grpck`). Then print their exit codes so I know if either one found a fatal error."

**Reading it left to right (line 1):**
- `sudo pwck -r` → "**p**ass**w**d **ch**ec**k**, `-r` = **r**ead-only. Scan `/etc/passwd` + `/etc/shadow` for structural problems, but don't try to fix anything."

**Reading it left to right (line 2):**
- `sudo grpck -r` → "**gr**ou**p ch**ec**k**, `-r` = read-only. Scan `/etc/group` + `/etc/gshadow`."

**Reading it left to right (line 3):**
- `echo "exit codes: pwck=$? grpck=$?"` → "print the exit codes."
- `$?` → "the exit code of the **previous** command. But since two commands ran before this echo, `$?` only reflects `grpck`. The `pwck=` value is also `grpck`'s exit code — a known limitation of this one-liner. (To capture both, you'd save them separately.)"

**The story:** `pwck` and `grpck` are the **hidden engines** behind `vipw`/`vigr` — those wrappers call these checkers automatically when you save. But you can also call them on demand to audit the database without opening an editor. The `-r` flag makes them strictly read-only — they only report issues, never modify. Exit code 0 = clean. Anything non-zero = problems. The typical "harmless" warnings are about service users with `/dev/null` homes or `/sbin/nologin` shells — those aren't bugs, they're intentional hardening.

**Commands**

| Token | Meaning |
|---|---|
| `pwck -r` | Read-only check (no fixes) |
| `pwck` (no `-r`) | Interactive — prompts to fix issues |
| `grpck -r` | Same for `/etc/group` / `/etc/gshadow` |

**Output decoded**

| Element | Meaning |
|---|---|
| Warnings (`directory not exist`) | Often harmless — service users use `/dev/null` |
| `no matching password file entry` | A shadow record without a passwd record — investigate |
| Exit code 0 | No fatal errors |

**Why a sysadmin needs this on RHCA RH342:** First diagnostic after suspected corruption.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `pwck` says "duplicate UID" | Two records share a UID — only one is canonical; reconcile carefully |

---

### Task 11 — Use `visudo` to add a user to sudoers

**Purpose:** Add a granular sudo rule via `visudo` (never edit `/etc/sudoers` directly).

```bash
sudo visudo
```

Inside (depending on your `$EDITOR`, this is `vi` by default):

```
G                         (jump to end)
o                         (open new line)
labtester ALL=(ALL) NOPASSWD: /bin/systemctl status sshd
Esc
:wq
```

`visudo` will syntax-check on save:

```
labtester  ALL=(ALL) NOPASSWD: /bin/systemctl status sshd
```

Verify:

```bash
sudo -l -U labtester
```

**Expected output:**

```
User labtester may run the following commands on this host:
    (ALL) NOPASSWD: /bin/systemctl status sshd
```

**Human-Readable Breakdown**

> "Open `/etc/sudoers` safely with `visudo`. Jump to the end of the file. Open a new line. Add a rule that lets `labtester` run `systemctl status sshd` as any user, with no password prompt. Save — `visudo` runs the sudoers parser before writing; if the syntax is wrong, it refuses to save. Then verify what `labtester` can actually do with `sudo -l -U`."

**Reading it left to right (the vi commands):**
- `sudo visudo` → "**vi sudo** — safe editor for `/etc/sudoers`. **Never** open this file with plain `vi`."
- `G` → "in vi: jump to the **last line** of the file."
- `o` → "in vi: **o**pen a new line below the current one and enter insert mode."
- `labtester ALL=(ALL) NOPASSWD: /bin/systemctl status sshd` → "the sudoers rule (see below)."
- `Esc` → "leave insert mode."
- `:wq` → "save. `visudo` runs the parser before writing."

**Reading the sudoers rule field by field:**
- `labtester` → "**who** the rule applies to (the user)."
- `ALL` → "**from which host** this rule is valid (`ALL` = any host this sudoers file is deployed on)."
- `=(ALL)` → "**run-as** target: `(ALL)` = can run the command as any user (typically root)."
- `NOPASSWD:` → "**tag** — skip the password prompt for this rule."
- `/bin/systemctl status sshd` → "**which command** is allowed. Must be the absolute path, exact match."

**Reading the verify line:**
- `sudo -l -U labtester` → "**list** sudo rules, `-U` = **for this user**. Shows what `labtester` is allowed to run."

**The story:** `visudo` is the gatekeeper that prevents the #1 self-inflicted system disaster: a broken `/etc/sudoers`. Plain `vi` will happily save a syntax error, and the moment you exit, every future `sudo` invocation will fail — including the `sudo vi` you'd need to fix it. `visudo` runs the sudoers grammar parser on save and refuses to write if it's broken, dropping you back into the editor with `e` / `x` / `Q` options. **Always** use `visudo` for sudoers, **never** edit `/etc/sudoers` directly.

**Commands**

| Token | Meaning |
|---|---|
| `sudo visudo` | Locked, syntax-checked edit |
| `USER HOST=(RUNAS) [TAG:] CMD` | Rule format |
| `NOPASSWD:` | Skip password prompt for this command |
| `sudo -l -U USER` | List USER's effective sudo rules |

**Output decoded**

| Element | Meaning |
|---|---|
| `(ALL)` | Can run as any user |
| `NOPASSWD:` | No password prompt |
| `/bin/systemctl status sshd` | Only this exact command |

**Why a sysadmin needs this on RHCSA EX200:** "Allow user X to run command Y without password" is a recurring exam task.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `>>> /etc/sudoers: syntax error near line N <<<` | `visudo` prompts for `e` (edit), `x` (exit and discard), `Q` (quit and write anyway — DANGEROUS) — choose `e` |
| `Defaults` rules cause weirdness | Cross-reference with `man sudoers` |

---

### Task 12 — Prefer drop-in files: `/etc/sudoers.d/`

**Purpose:** Modern best practice — never edit `/etc/sudoers` itself. Use small fragment files under `/etc/sudoers.d/`.

```bash
sudo visudo -f /etc/sudoers.d/labtester-svc
```

Inside:

```
i
labtester ALL=(ALL) NOPASSWD: /bin/systemctl restart sshd
Esc
:wq
sudo -l -U labtester
ls -l /etc/sudoers.d/labtester-svc
```

**Expected output (excerpt):**

```
-r--r-----. 1 root root  68 May 21 15:30 /etc/sudoers.d/labtester-svc
```

**Human-Readable Breakdown**

> "Don't touch `/etc/sudoers` directly. Instead, create a small fragment file in `/etc/sudoers.d/` — `visudo -f` opens that specific file with the same locking and syntax-checking. Add a rule giving `labtester` `NOPASSWD` access to restart sshd. Save. Verify."

**Reading it left to right:**
- `sudo visudo -f /etc/sudoers.d/labtester-svc` → "safe edit, `-f` = open this **specific file** instead of `/etc/sudoers`. The file will be created if it doesn't exist."
- `i` → "in vi: enter **i**nsert mode."
- `labtester ALL=(ALL) NOPASSWD: /bin/systemctl restart sshd` → "the rule (same format as Task 11)."
- `Esc` → "leave insert mode."
- `:wq` → "save — `visudo` parses the file before writing."
- `sudo -l -U labtester` → "list labtester's effective sudo rules. Should now include this restart command."
- `ls -l /etc/sudoers.d/labtester-svc` → "verify the file's permissions."

**Filename rules for `/etc/sudoers.d/`:**
- No `.` anywhere in the filename (sudo silently skips files with dots — including `.rpmnew`).
- No trailing `~` (skipped as backup files).
- Should match `[a-zA-Z0-9_-]+`.
- Must be mode `0440`, owner `root:root`.

**The story:** This is **the** modern best practice for sudo rules. The main `/etc/sudoers` file gets replaced by package upgrades (`.rpmnew`); your custom drop-ins under `/etc/sudoers.d/` survive. Each service or admin role gets its own fragment file (e.g., `labtester-svc`, `appops-deploy`, `monitoring-readonly`) — easier to audit, easier to roll back. `visudo -f` opens any specific file with the same locking + syntax check that protects `/etc/sudoers`.

**Commands**

| Token | Meaning |
|---|---|
| `visudo -f /etc/sudoers.d/NAME` | Lock + edit a drop-in fragment |
| Filename rules | No `.` in the name (sudo skips it); no underscores in some versions either |
| Permissions | Sudo enforces `0440` automatically |

**Output decoded**

| Element | Meaning |
|---|---|
| Mode `0440` | Read for owner + group, none for others |
| Owner root:root | Required |

**Why a sysadmin needs this on RHCA RH358:** Drop-ins survive package upgrades; main `/etc/sudoers` may get `.rpmnew`-replaced.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Rule ignored | Check filename: `sudo visudo -c -f /etc/sudoers.d/NAME` — fragment names must match `[a-zA-Z0-9_-]+` and **must not** end in `~` or contain `.` |

---

### Task 13 — Syntax-check `sudoers` without opening an editor

**Purpose:** Run a parser in CI / pre-commit hooks.

```bash
sudo visudo -c
sudo visudo -c -f /etc/sudoers.d/labtester-svc
```

**Expected output:**

```
/etc/sudoers: parsed OK
/etc/sudoers.d/labtester-svc: parsed OK
```

**Human-Readable Breakdown**

> "Run `visudo` in check-only mode — no editor opens. First check the main `/etc/sudoers` file. Then check the drop-in fragment from Task 12. Both should report `parsed OK`."

**Reading it left to right:**
- `sudo visudo -c` → "`visudo`, `-c` = **c**heck only. Parse `/etc/sudoers` (the default target) and exit. No editor. No edit. Just a syntax verdict."
- `sudo visudo -c -f /etc/sudoers.d/labtester-svc` → "`-c` = check, `-f FILE` = use this specific file instead of `/etc/sudoers`."

**The story:** `visudo -c` is the **CI/CD-friendly** way to validate sudoers without opening an editor. Use it in:
- **Ansible** — the `community.general.sudoers` module calls `visudo -cf` automatically via `validate:`.
- **Pre-commit hooks** — refuse to commit a broken sudoers fragment.
- **Configuration management pipelines** — gate every deploy on `visudo -c` exit code 0.
- **Manual rescue** — after a hand-edit, run `sudo visudo -c` to confirm before logging out.

The output format is simple: `path: parsed OK` (good) or `path: >>> error <<<` (bad). Exit code 0 = safe to deploy, non-zero = stop the pipeline.

**Commands**

| Flag | Meaning |
|---|---|
| `-c` | **C**heck only — no edit |
| `-c -f FILE` | Check specific file |

**Output decoded**

| Line | Meaning |
|---|---|
| `parsed OK` | Safe to deploy |
| `>>> error <<<` | Don't deploy — fix first |

**Why a sysadmin needs this on RHCE EX294:** Run as a Molecule / lint step before applying changes from playbooks.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Wrong sudo policy in CI | Run `-c` in your container image; treat non-zero as failure |

---

### Task 14 — Recover from a corrupted `/etc/passwd` (simulation)

**Purpose:** Practice the worst case — a malformed passwd line. **Do this on a disposable VM only.**

```bash
sudo cp /etc/passwd /etc/passwd.GOOD
# Simulate corruption (e.g., a missing colon)
sudo sed -i 's/^labtester:x:/labtesterxxx/' /etc/passwd
sudo pwck -r
echo "exit=$?"
```

`pwck` will report something like:

```
invalid password file entry
delete line 'labtesterxxx1002:1002:Lab27 Tester:/home/labtester:/bin/bash'? n
…
exit=2
```

Now recover:

```bash
sudo cp /etc/passwd.GOOD /etc/passwd
sudo pwck -r && echo "fixed"
```

**Human-Readable Breakdown**

> "Practice the worst-case scenario. Step 1: make a backup copy of `/etc/passwd` so you have something to restore. Step 2: deliberately break the labtester line with `sed` (remove the colon after the username). Step 3: run `pwck` to detect the corruption. Step 4: restore from the backup. Step 5: re-run `pwck` to confirm everything is healed."

**Reading it left to right (the corruption simulation):**
- `sudo cp /etc/passwd /etc/passwd.GOOD` → "**c**o**p**y `/etc/passwd` to `passwd.GOOD`. **Always back up first.**"
- `sudo sed -i 's/^labtester:x:/labtesterxxx/' /etc/passwd` → "edit the file in place."
  - `sed` → "**s**tream **ed**itor."
  - `-i` → "**i**n-place (modifies the file directly)."
  - `s/^labtester:x:/labtesterxxx/` → "**s**ubstitute. Replace `labtester:x:` (at start of line) with `labtesterxxx` — note the missing colons. This mangles the field structure."
- `sudo pwck -r` → "scan for corruption."
- `echo "exit=$?"` → "print the exit code. Non-zero means problems."

**Reading it left to right (the recovery):**
- `sudo cp /etc/passwd.GOOD /etc/passwd` → "restore from backup."
- `sudo pwck -r && echo "fixed"` → "re-verify; if `pwck` exits 0, print `fixed`."

**The story:** This task practices the **panic procedure** for corrupted identity files. The order matters: **backup → modify → verify → recover**. On the RHCA RH342 exam, you're expected to handle exactly this scenario — a script or a tired sysadmin breaks `/etc/passwd`, and you have minutes to restore it. The two recovery sources, in order of preference: (1) your manual backup (`/etc/passwd.GOOD`), (2) the auto-backup that `vipw` leaves at `/etc/passwd-` (covered in Task 15). **Only do this on a disposable VM** — if you break `/etc/passwd` on a machine where `sudo` is your only path to root, you may need single-user mode to recover.

**Commands**

| Token | Meaning |
|---|---|
| `cp /etc/passwd /etc/passwd.GOOD` | Backup BEFORE you touch anything |
| `pwck -r` | Detect the corruption |
| Restore via `cp` | The reverse |
| Final `pwck -r` | Confirm |

**Output decoded**

| Element | Meaning |
|---|---|
| Detected bad line | Demonstrates `pwck`'s value |
| Exit code 2 | Non-trivial errors found |
| `fixed` | After restore |

**Why a sysadmin needs this on RHCA RH342:** Real incidents — a buggy provisioning script corrupts the database. You must be calm and methodical.

> ⚠️ **Only run this on a VM you can reinstall.** If your only `root` session is the corrupt one, you may be unable to recover without single-user mode.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `sudo` doesn't work after corruption | Reboot into single-user / rescue mode; mount filesystem; restore from backup |
| No backup available | `cp /etc/passwd- /etc/passwd` — `passwd-` is the previous version automatically kept by `vipw`/`shadow-utils` |

---

### Task 15 — Use the `*-` backup files

**Purpose:** `vipw`/`vigr` automatically write a previous-version `*-` file. Lifesaver during recovery.

```bash
ls -l /etc/passwd /etc/passwd- /etc/shadow /etc/shadow- /etc/group /etc/group- /etc/gshadow /etc/gshadow- 2>/dev/null
sudo diff /etc/passwd /etc/passwd- | head
```

**Expected output (excerpt):**

```
-rw-r--r--. 1 root root  2876 May 21 15:32 /etc/passwd
-rw-r--r--. 1 root root  2785 May 21 15:25 /etc/passwd-
-rw-------. 1 root root  1432 May 21 15:32 /etc/shadow
-rw-------. 1 root root  1402 May 21 15:25 /etc/shadow-
…
14a15
> labtester:x:1002:1002:Lab27 Tester:/home/labtester:/bin/bash
```

**Human-Readable Breakdown**

> "Show me the long listing of all four identity files AND their auto-backup siblings (the `*-` files that `vipw`/`vigr` keep automatically). Then `diff` the live `/etc/passwd` against its backup to see exactly what changed since the last successful edit."

**Reading it left to right (line 1):**
- `ls -l` → "long listing of all these paths."
- `/etc/passwd /etc/passwd- /etc/shadow /etc/shadow- /etc/group /etc/group- /etc/gshadow /etc/gshadow-` → "the four live files and their four `-` backup siblings."
- `2>/dev/null` → "if any file doesn't exist (e.g., `/etc/gshadow-` on a fresh system), redirect that error message (port 2 = stderr) to the trash."

**Reading it left to right (line 2):**
- `sudo diff /etc/passwd /etc/passwd-` → "show the **diff**erences between the live file and its backup."
- `| head` → "show only the first 10 lines (default `head` count)."

**The story:** Every time you save a successful edit via `vipw`/`vigr`, the previous version of the file is renamed with a trailing `-`. This gives you a **free undo**: if your latest edit was wrong, `sudo cp /etc/passwd- /etc/passwd` restores the previous good version. The diff output makes change auditing easy — `diff /etc/passwd /etc/passwd-` shows exactly which lines were added or removed in the most recent edit. On RHCA RH342, when you arrive at a corrupted system, **first check `/etc/passwd-`**, **then** check off-host backups. The `-` files are your safety net.

**Commands**

| Token | Meaning |
|---|---|
| `/etc/passwd-` | Auto-backup of the previous version |
| `/etc/shadow-` | Same for shadow |
| `/etc/group-` / `/etc/gshadow-` | Same for group databases |

**Output decoded**

| Element | Meaning |
|---|---|
| Two files exist | Backups are present |
| `diff` shows what's new since the last successful edit | Real change log |

**Why a sysadmin needs this on RHCA RH342:** Free undo — restore is `cp /etc/passwd- /etc/passwd` (verify with `pwck -r` after).

**Troubleshoot**

| Symptom | Fix |
|---|---|
| No `-` files | Older tooling or recent fresh install — no edits yet |
| Backup is also corrupt | Restore from off-host backup or rebuild from `useradd` |

---

### Task 16 — Audit account status: `passwd -S`, `chage -l`

**Purpose:** Read state from shadow without editing.

```bash
sudo passwd -S labtester
sudo chage -l labtester
sudo awk -F: '$2 ~ /^!/ { print "LOCKED:", $1 }' /etc/shadow | head
```

**Expected output (excerpt):**

```
labtester PS 2026-05-21 1 60 7 -1 (Password set, SHA512 crypt.)
Last password change                                : May 21, 2026
Password expires                                    : Jul 20, 2026
…
LOCKED: root        (example)
LOCKED: bin
```

**Human-Readable Breakdown**

> "Three different ways to inspect account state without editing anything. First: a one-line status for `labtester`. Second: aging details for `labtester`. Third: scan all of `/etc/shadow` and print the username of every account whose password hash starts with `!` (i.e., locked)."

**Reading it left to right (line 1):**
- `sudo passwd -S labtester` → "**passwd**, `-S` = **S**tatus. Print a single line: `<user> <state> <last-change> <min> <max> <warn> <inactive> (hash type)`."

**Reading it left to right (line 2):**
- `sudo chage -l labtester` → "**ch**ange **age**, `-l` = list. Multi-line aging report."

**Reading it left to right (line 3):**
- `sudo awk -F: '$2 ~ /^!/ { print "LOCKED:", $1 }' /etc/shadow` → "scan all of `/etc/shadow`."
  - `-F:` → "**F**ield separator = `:`."
  - `$2 ~ /^!/` → "**if** field 2 (the password hash) matches the pattern `^!` (starts with `!`)."
  - `~` → "regex-match operator."
  - `^!` → "anchored regex: line starts with `!`."
  - `{ print "LOCKED:", $1 }` → "print the literal text `LOCKED:` followed by field 1 (the username)."
- `| head` → "show only the first 10 results."

**Status letters from `passwd -S`:**
- `PS` → "**P**assword **S**et."
- `L` → "**L**ocked (`!` prefix on hash)."
- `NP` → "**N**o **P**assword (empty field 2 — dangerous!)."
- `LK` → "**L**oc**K**ed and has no real password."

**The story:** Reading account state is a **read-only** workflow — you should never need to edit `/etc/shadow` to figure out who's locked or when a password expires. `passwd -S` gives the compact one-liner. `chage -l` gives the verbose report. And the `awk` pattern is the bulk-audit pattern — "show me every locked account" — which on RHCA RH358 service-account hygiene reviews is the very first question.

**Commands**

| Token | Meaning |
|---|---|
| `passwd -S USER` | One-line account status |
| `chage -l USER` | Aging details |
| `awk` over `/etc/shadow` | Bulk audit of locked accounts |

**Output decoded**

| Letter | Meaning |
|---|---|
| `PS` | Password set |
| `L` | Locked |
| `NP` | No password |
| `LK` | Locked + no password |

**Why a sysadmin needs this on RHCA RH358:** Service-account hygiene — confirm every `nologin` account is also `passwd -l`'d.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Unexpected `NP` | Account has empty password — fix with `passwd USER` or `passwd -l USER` |

---

### Task 17 — Lock and unlock an account

**Purpose:** Disable login without deleting the user.

```bash
sudo passwd -l labtester
sudo passwd -S labtester
sudo passwd -u labtester
sudo passwd -S labtester
```

**Expected output:**

```
labtester LK 2026-05-21 1 60 7 -1 (Password locked.)
labtester PS 2026-05-21 1 60 7 -1 (Password set, SHA512 crypt.)
```

**Human-Readable Breakdown**

> "Lock `labtester`'s password — prepends `!` to the hash so no password will ever match. Verify the locked state. Unlock it. Verify the unlocked state."

**Reading it left to right:**
- `sudo passwd -l labtester` → "**passwd**, `-l` = **l**ock. Prepends `!` to the hash in `/etc/shadow` field 2."
- `sudo passwd -S labtester` → "show status — should report `LK` (locked)."
- `sudo passwd -u labtester` → "**u**nlock — removes the `!` prefix."
- `sudo passwd -S labtester` → "show status — should report `PS` (password set)."

**The story:** Locking a password is the non-destructive way to disable a login without deleting the user (which would lose their files and ownership). It works by prepending `!` to the stored hash, which makes the hash impossible to match against any input. **Important gotcha:** `passwd -l` only blocks **password-based** login. If the user has an SSH key in `~/.ssh/authorized_keys`, they can still log in. The complete "disable account" recipe is: `passwd -l USER && chsh -s /sbin/nologin USER && (optionally) rm ~USER/.ssh/authorized_keys`. On RHCSA, "disable login for user X" is a recurring task — `passwd -l` is the answer.

**Commands**

| Token | Meaning |
|---|---|
| `passwd -l USER` | Lock (prepend `!` to hash in `/etc/shadow`) |
| `passwd -u USER` | Unlock |
| `usermod -L USER` / `-U USER` | Same as above (alternative) |

**Output decoded**

| Field | Meaning |
|---|---|
| `LK` → `PS` | State transitions |

**Why a sysadmin needs this on RHCSA EX200:** Common task: "Disable user X's password login." `passwd -l` is the answer.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| User can still SSH | They may use a key — also remove `~/.ssh/authorized_keys` or `usermod -s /sbin/nologin` |

---

### Task 18 — Edit `/etc/passwd` to change a shell safely

**Purpose:** `chsh` is preferred, but show that `vipw` works for shells too.

```bash
which nologin
sudo vipw
```

Inside:

```
/^labtester:
:s|:/bin/bash$|:/sbin/nologin|
:wq
getent passwd labtester
```

**Expected output:**

```
labtester:x:1002:1002:Updated GECOS Field:/home/labtester:/sbin/nologin
```

Revert (in production, you'd use `chsh -s /bin/bash labtester`):

```bash
sudo chsh -s /bin/bash labtester
getent passwd labtester
```

**Human-Readable Breakdown**

> "First confirm `nologin`'s path. Then open `/etc/passwd` safely with `vipw`. Find `labtester`'s line. Replace the shell field (the last colon-delimited field) with `/sbin/nologin` using a vi substitute, anchored to end-of-line. Save. Verify. Then revert using `chsh` (the standard tool)."

**Reading it left to right:**
- `which nologin` → "show me the path to `nologin` (typically `/sbin/nologin`)."
- `sudo vipw` → "open `/etc/passwd` safely."
- `/^labtester:` → "find labtester's line."
- `:s|:/bin/bash$|:/sbin/nologin|` → "substitute on current line."
  - `s` → "**s**ubstitute."
  - `|` → "delimiter (using `|` instead of `/` so the slashes in shell paths don't conflict)."
  - `:/bin/bash$` → "search pattern: a colon, then `/bin/bash`, then `$` (end-of-line anchor — ensures we only match the shell field, not a path elsewhere)."
  - `:/sbin/nologin` → "replacement: a colon, then the new shell path."
- `:wq` → "save. `vipw` runs `pwck` before writing."
- `getent passwd labtester` → "verify via NSS."
- `sudo chsh -s /bin/bash labtester` → "the **standard** way to change a shell. `chsh -s SHELL USER`."

**The story:** `chsh` is the right tool — it handles validation (the shell must be listed in `/etc/shells`), permissions, and the file update for you. The `vipw` substitution shown here is the **recovery** method, used when `chsh` can't be invoked (e.g., it's broken, or you need to fix many users at once via a script). The end-of-line anchor `$` is the key safety feature in the regex — without it, you could accidentally replace `/bin/bash` somewhere in the GECOS field or another column. **Service accounts** should always have `/sbin/nologin` as their shell — quick audit: `awk -F: '$NF !~ /(nologin|false)$/ { print $1 }' /etc/passwd` lists every non-`nologin` account.

**Commands**

| Token | Meaning |
|---|---|
| `:s\|A\|B\|` | Substitute on current line; `\|` delim avoids `/` conflict |
| `:/bin/bash$` | End-of-line anchor — only the shell field |
| `chsh -s SHELL USER` | Standard tool |

**Output decoded**

| Element | Meaning |
|---|---|
| Shell now `/sbin/nologin` | User can't log in interactively |
| After revert | Back to bash |

**Why a sysadmin needs this:** Service accounts should always have `/sbin/nologin` — quick audits via `awk -F: '$NF !~ /(false|nologin)$/ { print $1 }' /etc/passwd`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Shell not in `/etc/shells` | Add it with `sudo chsh --list-shells`; `chsh` is strict |

---

### Task 19 — Build a comprehensive identity audit report

**Purpose:** Combine everything — produce a one-page audit of users, groups, sudo rules, locked accounts.

```bash
{
  echo "=== Identity Audit — $(date -Iseconds) on $(hostname) ==="
  echo

  echo "--- Regular users (UID >= 1000) ---"
  awk -F: '$3 >= 1000 && $3 < 60000 { printf "%-20s UID=%-6d GID=%-6d shell=%s\n", $1, $3, $4, $7 }' /etc/passwd

  echo
  echo "--- Locked accounts ---"
  sudo awk -F: '$2 ~ /^[!*]/ { print $1 }' /etc/shadow

  echo
  echo "--- Groups containing labtester ---"
  id labtester 2>/dev/null
  getent group | awk -F: -v u="labtester" '$4 ~ "(^|,)" u "(,|$)" { print $1 }'

  echo
  echo "--- sudoers main + drop-ins ---"
  sudo grep -hE '^[^#].*ALL' /etc/sudoers /etc/sudoers.d/* 2>/dev/null

  echo
  echo "--- Syntax checks ---"
  sudo pwck -r >/dev/null 2>&1   && echo "pwck:    OK"   || echo "pwck:    ISSUE"
  sudo grpck -r >/dev/null 2>&1  && echo "grpck:   OK"   || echo "grpck:   ISSUE"
  sudo visudo -c >/dev/null 2>&1 && echo "sudoers: OK"   || echo "sudoers: ISSUE"
} > ~/identity-audit.txt

cat ~/identity-audit.txt
```

**Expected output (excerpt):**

```
=== Identity Audit — 2026-05-21T15:40:00+00:00 on ip-172-31-45-12.ec2.internal ===

--- Regular users (UID >= 1000) ---
ec2-user             UID=1000   GID=1000   shell=/bin/bash
labtester            UID=1002   GID=1002   shell=/bin/bash
teststaff            UID=1003   GID=1003   shell=/bin/bash

--- Locked accounts ---
bin
daemon
…

--- Groups containing labtester ---
uid=1002(labtester) gid=1002(labtester) groups=1002(labtester),1500(developers)
developers

--- sudoers main + drop-ins ---
root    ALL=(ALL)       ALL
labtester ALL=(ALL) NOPASSWD: /bin/systemctl restart sshd

--- Syntax checks ---
pwck:    OK
grpck:   OK
sudoers: OK
```

**Human-Readable Breakdown**

> "Build a one-page identity audit report by running five separate diagnostics, capturing all their output into a file. The whole thing is wrapped in `{ ... } > ~/identity-audit.txt` — that's a brace group; everything inside writes to the same file."

**Reading the wrapping syntax:**
- `{ ... }` → "**brace group** — runs multiple commands as a unit. Unlike `( ... )` which creates a subshell, `{ }` runs in the current shell."
- `> ~/identity-audit.txt` → "redirect all stdout from the entire group into this file (overwriting it)."

**Reading each section, top to bottom:**
- `echo "=== Identity Audit — $(date -Iseconds) on $(hostname) ==="` → "header. `$(...)` runs commands inline; `date -Iseconds` = ISO-8601 timestamp."
- `awk -F: '$3 >= 1000 && $3 < 60000 { printf "%-20s UID=%-6d GID=%-6d shell=%s\n", $1, $3, $4, $7 }' /etc/passwd` → "**list regular human users**."
  - `-F:` → "split on `:`."
  - `$3 >= 1000 && $3 < 60000` → "**condition**: UID is in the human range (system users are < 1000; `nobody` is 65534)."
  - `{ printf ... }` → "if condition is true, print formatted output. `%-20s` = left-justified 20-char string. `%-6d` = left-justified 6-digit integer."
- `sudo awk -F: '$2 ~ /^[!*]/ { print $1 }' /etc/shadow` → "**list locked accounts**: field 2 starts with `!` or `*`."
- `id labtester 2>/dev/null` → "**show labtester's groups** (silently ignore if user doesn't exist)."
- `getent group | awk -F: -v u="labtester" '$4 ~ "(^|,)" u "(,|$)" { print $1 }'` → "**find every group containing labtester**."
  - `-v u="labtester"` → "pass a shell variable into awk as `u`."
  - `$4 ~ "(^|,)" u "(,|$)"` → "field 4 (members) matches: start-of-string-or-comma, then the username, then comma-or-end. This avoids false positives like `labtester2` matching `labtester`."
- `sudo grep -hE '^[^#].*ALL' /etc/sudoers /etc/sudoers.d/* 2>/dev/null` → "**list active sudo rules**."
  - `-h` → "**h**ide filename prefix in output."
  - `-E` → "**E**xtended regex."
  - `^[^#].*ALL` → "lines that don't start with `#` (skip comments) and contain `ALL`."
- `sudo pwck -r >/dev/null 2>&1 && echo "pwck: OK" || echo "pwck: ISSUE"` → "**syntax check** the user DB."
  - `>/dev/null 2>&1` → "throw away both stdout and stderr — only the exit code matters."
  - `&& echo OK || echo ISSUE` → "if exit code is 0, print OK; otherwise print ISSUE."

**The story:** This is the **compliance one-pager** every team needs. It answers five questions in one shot: Who are the human users? Which accounts are locked? Which groups does this user belong to? What sudo rules are live? Are the databases structurally sound? In an audit interview ("show me your identity hygiene"), the answer is `cat ~/identity-audit.txt`. On RHCA RH358 service-account reviews, this is a monthly recurring task — automate it with a cron job.

**Commands**

| Token | Meaning |
|---|---|
| `awk -F: '$3 >= 1000 && $3 < 60000'` | Regular humans |
| `awk -F: '$2 ~ /^[!*]/'` | Locked-by-password-hash accounts |
| `getent group \| awk` | Find groups including a user |
| `grep -hE '^[^#]'` | Active rules only |
| `pwck -r`, `grpck -r`, `visudo -c` | Three sanity checks |

**Output decoded**

| Section | Meaning |
|---|---|
| Regular users | Auditable accounts |
| Locked accounts | Hardened service users |
| Groups containing user | Cross-reference |
| sudoers active rules | Privilege baseline |
| Syntax checks | Trust signal |

**Why a sysadmin needs this on RHCA RH358:** Monthly compliance report.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Empty sections | Audit script run on a fresh host — expected |

---

### Task 20 — Exam-style scenario: end-to-end safe identity change

**Task statement (RHCSA / RH358-style):** *"Create user `appops` with UID 2001, primary group `appops` (GID 2001), supplementary group `developers`, home directory `/srv/appops`, shell `/bin/bash`. Give `appops` `NOPASSWD` sudo to `/bin/systemctl restart httpd`. Set the password to expire in 30 days. Verify every change. Provide a unified diff of `/etc/passwd` before and after."*

```bash
# 1. Snapshot before
sudo cp /etc/passwd /tmp/passwd.before
sudo cp /etc/group  /tmp/group.before

# 2. Create group with explicit GID
sudo groupadd -g 2001 appops

# 3. Create user with explicit UID, home, shell, gecos
sudo useradd -u 2001 -g appops -G developers -d /srv/appops -m -s /bin/bash -c "App Operator" appops

# 4. Initial password (in real exam: use a known value)
echo 'appops:TempPa55!' | sudo chpasswd

# 5. Aging
sudo chage -M 30 -W 7 appops

# 6. Sudo drop-in
sudo bash -c 'cat > /etc/sudoers.d/appops <<EOF
appops ALL=(ALL) NOPASSWD: /bin/systemctl restart httpd
EOF'
sudo chmod 0440 /etc/sudoers.d/appops

# 7. Validate
sudo visudo -c -f /etc/sudoers.d/appops
sudo pwck -r
sudo grpck -r
id appops
getent passwd appops
getent group appops developers
sudo chage -l appops
sudo passwd -S appops
sudo -l -U appops

# 8. Diff /etc/passwd before/after
sudo diff -u /tmp/passwd.before /etc/passwd

# 9. Cleanup recommendation (optional; do NOT run in real exam)
# sudo userdel -r appops && sudo groupdel appops && sudo rm -f /etc/sudoers.d/appops
```

**Expected output (excerpt):**

```
/etc/sudoers.d/appops: parsed OK
uid=2001(appops) gid=2001(appops) groups=2001(appops),1500(developers)
appops:x:2001:2001:App Operator:/srv/appops:/bin/bash
appops:x:2001:
developers:x:1500:labtester,teststaff
Last password change                                : May 21, 2026
Password expires                                    : Jun 20, 2026
Number of days of warning before password expires   : 7
appops PS 2026-05-21 0 30 7 -1 (Password set, SHA512 crypt.)
User appops may run the following commands on this host:
    (ALL) NOPASSWD: /bin/systemctl restart httpd
--- /tmp/passwd.before
+++ /etc/passwd
@@ -34,0 +35,1 @@
+appops:x:2001:2001:App Operator:/srv/appops:/bin/bash
```

**Human-Readable Breakdown**

> "End-to-end safe identity change with full audit. Snapshot `/etc/passwd` and `/etc/group` first. Create the `appops` group with GID 2001. Create the `appops` user with all attributes set in one `useradd` call. Set the password non-interactively. Set aging policy. Drop a sudoers fragment. Validate everything with `visudo -c`, `pwck -r`, and `grpck -r`. Cross-verify the result from five different angles. Finally, produce a unified diff of `/etc/passwd` before vs. after for the audit log."

**Reading it step by step:**

**Step 1 — Snapshots**
- `sudo cp /etc/passwd /tmp/passwd.before` → "**c**o**p**y the live file to a 'before' snapshot. Same for `/etc/group`. This is your audit baseline."

**Step 2 — Create group with explicit GID**
- `sudo groupadd -g 2001 appops` → "**add** a **group** named `appops`."
- `-g 2001` → "explicitly pin **g**roup ID to 2001. Without this, the system picks the next free GID."

**Step 3 — Create user with everything in one shot**
- `sudo useradd ... appops` → "add the user."
- `-u 2001` → "**u**ser ID = 2001."
- `-g appops` → "**g**roup (primary) = `appops`."
- `-G developers` → "**G**roups (supplementary, comma-separated list) = `developers`."
- `-d /srv/appops` → "**d**irectory (home) = `/srv/appops`."
- `-m` → "**m**ake the home directory."
- `-s /bin/bash` → "**s**hell."
- `-c "App Operator"` → "**c**omment (GECOS) field."

**Step 4 — Non-interactive password set**
- `echo 'appops:TempPa55!' | sudo chpasswd` → "**ch**ange **passwd** in bulk."
- `chpasswd` reads `user:password` pairs from stdin — used in scripts to avoid the interactive `passwd` prompt.

**Step 5 — Aging**
- `sudo chage -M 30 -W 7 appops` → "**ch**ange **age**: **M**ax 30 days password validity, **W**arn 7 days before expiry."

**Step 6 — Sudo drop-in**
- `sudo bash -c 'cat > /etc/sudoers.d/appops <<EOF ... EOF'` → "use a here-doc to create the fragment file as root."
- `cat > file <<EOF ... EOF` → "**here-doc** pattern: send the lines between `<<EOF` and `EOF` to `cat`'s stdin, which writes them to the file."
- `sudo chmod 0440 /etc/sudoers.d/appops` → "set the required sudoers fragment permissions (read for owner+group, none for others)."

**Step 7 — Validate everything**
- `sudo visudo -c -f /etc/sudoers.d/appops` → "syntax-check the sudoers fragment."
- `sudo pwck -r` → "verify user DB integrity."
- `sudo grpck -r` → "verify group DB integrity."
- `id appops` → "kernel-level identity view."
- `getent passwd appops` → "NSS lookup of the passwd entry."
- `getent group appops developers` → "NSS lookup of both groups."
- `sudo chage -l appops` → "aging report."
- `sudo passwd -S appops` → "one-line account status."
- `sudo -l -U appops` → "sudo privilege scope."

**Step 8 — Audit diff**
- `sudo diff -u /tmp/passwd.before /etc/passwd` → "**diff**erences in **u**nified format. Shows the exact line added — perfect for the audit log."

**The story:** This is a real RHCSA / RH358-style task condensed into one workflow. The pattern — **snapshot → change → multi-angle verify → diff for audit** — is what separates a junior admin from a senior one. The "snapshot first, diff after" habit means you can always answer "what did you change?" with a literal patch. The multi-angle verification (5+ different commands looking at the same change from different layers — kernel `id`, NSS `getent`, aging `chage`, status `passwd`, privilege `sudo -l`) ensures nothing slipped past you. Memorize this sequence; the exam grader is checking that you used the **right** tools.

**Step-by-step rationale**

| Step | Why |
|---|---|
| `cp` snapshots before changes | Diff target |
| `groupadd -g` | Pin GID so it matches the user's primary GID |
| `useradd -u … -g … -G … -d … -m -s … -c …` | Single command sets everything correctly |
| `chpasswd` | Non-interactive password set (`echo USER:PASS \| chpasswd`) |
| `chage -M 30 -W 7` | Aging policy |
| Drop-in sudoers + 0440 perms | Best-practice sudoers |
| `visudo -c -f` | Syntax-check the fragment |
| `pwck`/`grpck` | Confirm DB integrity |
| `id`, `getent`, `chage -l`, `passwd -S`, `sudo -l -U` | Full multi-angle verification |
| `diff -u` | Audit artifact |

**Output decoded**

| Element | Meaning |
|---|---|
| `parsed OK` | Sudo rule is safe to live |
| `id`/`getent` outputs | User and groups correctly set |
| `chage -l` | Aging policy in place |
| `passwd -S … PS … 30 7` | Aging fields visible at a glance |
| `sudo -l -U appops` | Privilege scope verified |
| Unified diff | Audit trail of the `/etc/passwd` change |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `useradd: UID 2001 is not unique` | Pick another UID or `userdel` the conflicting account first |
| `visudo -c` fails | Open with `sudo visudo -f /etc/sudoers.d/appops` and fix the syntax |
| Need to roll back | `sudo userdel -r appops && sudo groupdel appops && sudo rm -f /etc/sudoers.d/appops` |
| Lost sudo while testing | Use a second SSH session (which you opened **before** the change) to fix |

---

## 🔍 Safe-Edit Decision Guide

```
Edit /etc/passwd                       → sudo vipw          (or use useradd/usermod)
Edit /etc/shadow                       → sudo vipw -s       (or use chage/passwd)
Edit /etc/group                        → sudo vigr          (or use groupadd/gpasswd)
Edit /etc/gshadow                      → sudo vigr -s       (or use gpasswd)
Edit /etc/sudoers (main)               → sudo visudo
Edit /etc/sudoers.d/* (drop-ins)       → sudo visudo -f /etc/sudoers.d/NAME
Syntax-check sudoers without editing   → sudo visudo -c [-f FILE]
Integrity-check passwd/shadow          → sudo pwck -r
Integrity-check group/gshadow          → sudo grpck -r
Recovery from corruption               → restore /etc/<file>-  (auto-kept by vipw/vigr)
Lock account                           → sudo passwd -l USER    (or usermod -L)
Unlock account                         → sudo passwd -u USER    (or usermod -U)
Set aging policy                       → sudo chage -m N -M N -W N USER
Inspect status                         → passwd -S USER  /  chage -l USER  /  sudo -l -U USER
```

---

## ✅ Lab Checklist (20 Tasks)

- [ ] 01 Inspect `/etc/passwd` structure (field count = 7)
- [ ] 02 Inspect `/etc/shadow` structure (field count = 9)
- [ ] 03 Open `vipw`, leave with `:q!` to learn the workflow
- [ ] 04 Create a user with `useradd` and audit via `vipw`
- [ ] 05 Change GECOS via `vipw`
- [ ] 06 Set aging with `chage`, view shadow via `vipw -s`
- [ ] 07 Create a group via `groupadd`, audit via `vigr`
- [ ] 08 Add a group member via `vigr`
- [ ] 09 Inspect `/etc/gshadow` via `vigr -s`
- [ ] 10 Run `pwck -r` and `grpck -r`
- [ ] 11 Add a sudo rule via `visudo`
- [ ] 12 Prefer `/etc/sudoers.d/` drop-ins
- [ ] 13 `visudo -c` for syntax-only checks
- [ ] 14 Simulate corruption + recover (VM only)
- [ ] 15 Use auto-backup `*-` files
- [ ] 16 Audit account status with `passwd -S`, `chage -l`
- [ ] 17 Lock and unlock accounts
- [ ] 18 Change shell via `vipw` (and via `chsh`)
- [ ] 19 Build a multi-section identity audit
- [ ] 20 Exam combo: create-user + aging + sudo + verify + diff

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Editing `/etc/passwd` with `vi` directly | No lock, no integrity check, prone to corruption | Always use `sudo vipw` |
| Editing `/etc/sudoers` with `vi` directly | Broken sudo → no admin access | Always use `sudo visudo` |
| Adding `.` to a sudoers.d filename | Rule silently ignored | Rename without `.` (use letters, digits, `_`, `-`) |
| Forgetting `0440` perms on `/etc/sudoers.d/*` | sudo refuses to load | `sudo chmod 0440 /etc/sudoers.d/NAME` |
| Hand-editing `/etc/passwd` UID twice | Duplicate UIDs | `pwck -r` to detect, then `usermod -u` to fix |
| Missing colon | Field-count corruption | Restore from `*-` backup, fix the line in `vipw` |
| Using `passwd -l` and expecting no SSH | Key-based SSH still works | Also disable shell: `chsh -s /sbin/nologin USER` |
| Locking root via wrong sudoers | Stuck out of admin | Reboot to single-user; restore `/etc/sudoers` from backup |
| Missing `--check` step in CI | Broken sudo rolled out | Add `sudo visudo -c -f FILE` to your pipeline |
| Forgetting to re-login after group change | `id` shows old groups | Open a new session or use `newgrp`/`sg` |

---

## 📌 Exam Strategy

**RHCSA EX200**
- Memorize: `useradd`, `usermod`, `userdel`, `groupadd`, `groupmod`, `groupdel`, `chage`, `passwd`, `chsh`, `chfn`, `gpasswd` — those are the **right** tools.
- Use `vipw`/`vigr`/`visudo` for the rare cases the exam asks for manual verification or recovery.
- Always `pwck -r && grpck -r && sudo visudo -c` after a batch of identity changes.

**RHCE EX294 (Ansible)**
- `ansible.builtin.user`, `ansible.builtin.group`, `community.general.sudoers` (or `lineinfile` on `/etc/sudoers.d/*` with `validate: 'visudo -cf %s'`).
- Always `validate:` sudoers fragments so a bad template doesn't deploy.

**CKA**
- Linux node identity matters for kubelet user contexts, NFS mounts to pods, and PV ownership.
- Same `useradd`/`chage`/`visudo` skills apply to the underlying nodes.

**RHCA**
- RH342: Practice "I corrupted /etc/passwd, fix it in 5 minutes" drills.
- RH358: Service accounts — locked, `nologin` shell, minimal groups, no sudo.
- RH236: GlusterFS / NFS UID mapping — manually edit `/etc/passwd` and `/etc/group` consistently across cluster nodes.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 19 — `cat` | `cat /etc/passwd` for a quick read (don't `cat -A` and assume you understand the format) |
| Lab 22 — `grep` | `grep -E '^user:' /etc/passwd` for surgical lookups |
| Lab 23 — `diff` | Diff `/etc/passwd` vs `/etc/passwd-` to see your last change |
| Lab 24 — `sed` | `sed -i` is **NOT** safe for these files — always `vipw`/`vigr`/`visudo` |
| Lab 26 — `vi` | The editor `vipw`/`vigr`/`visudo` invoke — your `vi` skills carry over |

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
