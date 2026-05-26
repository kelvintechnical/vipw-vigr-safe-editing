# Lab: Safely Editing System Databases — `vipw`, `vigr`

**Series:** linux-ops-mastery — RHCSA Text File Management
**Subjects covered:** `vipw` and `vipw -s` (shadow), `vigr` and `vigr -s` (gshadow), why locked editing matters for `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow`, the `.lock` files (`/etc/passwd.lock`, `/etc/shadow.lock`, `/etc/group.lock`, `/etc/gshadow.lock`) and `passwd.OLD`/`shadow.OLD` backups, the field format of each database, choosing `$EDITOR` (`EDITOR=vi vipw`), `pwck` / `grpck` consistency checks, `pwconv` / `pwunconv` / `grpconv` / `grpunconv`, `visudo` as a cousin for `/etc/sudoers`, and how Ansible's `user`/`group` modules wrap these tools
**Career arcs covered:** RHCSA (the *required* way to edit user/group databases on the exam), RHCE (managing users at scale with `ansible.builtin.user`/`group`), SRE (auditing and rolling back unsafe `/etc/passwd` edits), DevOps (CI pipelines that rotate service accounts), AI/MLOps (managing shared GPU-cluster accounts)
**Prerequisite:** Lab 26 (vi/vim), Lab 22 (regex if you're using `:%s///`)
**Time Estimate:** 30 to 40 minutes
**Difficulty arc:** Task 1 foundation (what locked editing means) · 2 `vipw` walkthrough · 3 `vipw -s` for shadow · 4 `vigr` and `vigr -s` · 5 `pwck`/`grpck` audits · 6 RHCSA exam-realistic capstone

---

## Objective

Edit `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow` *safely* — meaning under a file lock that prevents another process (or `useradd` / `passwd` / `chage`) from corrupting the file at the same time. By the end of this lab you can switch a user's shell, change a group's GID, and recover from a syntactically broken edit.

The capstone is an exam-realistic prompt: *"Using `vipw`, change a test user's login shell to `/sbin/nologin`. Using `vipw -s`, set that user's password-aging max-days to 30. Using `vigr`, add the user to the `wheel` group. Run `pwck` and `grpck` to confirm consistency."*

> **Lab safety note:** All operations use a throwaway user `vipwtest`. We never edit `root`'s entry. Backups under `/root/vipw-backups/` are taken before any change so you can roll back.

---

## Concept: Locked Editing Prevents Corruption

`/etc/passwd` and friends are touched by many tools (`useradd`, `usermod`, `passwd`, `chsh`, `chage`, `chfn`, PAM, `systemd-userdbd`, etc.). If two of those write at the same time as `vim`, the file can end up half-written and unbootable.

`vipw` solves this by:

1. **Acquiring `/etc/passwd.lock`** (a flock-style sentinel) so no other tool can write.
2. **Opening the file in `$EDITOR`** (defaults to `vi`).
3. **Syntactically checking** the result on save.
4. **Rotating the previous version to `/etc/passwd.OLD`** as an automatic backup.
5. **Releasing the lock** when the editor exits.

```
   Editor running: vipw acquires /etc/passwd.lock
              │
              ▼
   useradd / passwd: tries to acquire .lock → blocks until vipw releases
              │
              ▼
   On save:   passwd → passwd.OLD (backup)
              new contents → passwd
              syntax check
              release lock
```

`vigr` does the same for `/etc/group`. `vipw -s` does the same for `/etc/shadow`. `vigr -s` does the same for `/etc/gshadow`.

> **Why this matters:** A botched `/etc/passwd` with a missing field can lock everyone out, including root, leading to a recovery-mode reboot. `vipw` is the safe net the exam expects you to use.

---

## 📜 Why `vipw` / `vigr` Exist — The Story

The original Unix had `vipasswd` (a one-line shell wrapper around `vi`) on V7 (1979). By the 1990s the **shadow-utils** project on Linux generalized it to `vipw` (passwd), `vipw -s` (shadow), `vigr` (group), and `vigr -s` (gshadow). Karel Zak and the util-linux maintainers added the locking and OLD-backup semantics.

`visudo` (Todd Miller, 1994) followed the same idea for `/etc/sudoers`: parse-check before saving so a typo doesn't lock you out of `sudo`.

> **The point of the story:** Anywhere a kernel/security database is shared between a human editor and a privileged daemon, a "vi-wrapped, locked, parse-checked, backup-on-save" tool exists. Use it.

---

## 👪 The Safe-Edit Family — Who Lives There

| Tool | Edits | Lock file | Backup |
|---|---|---|---|
| `vipw` | `/etc/passwd` | `/etc/passwd.lock` | `/etc/passwd.OLD` |
| `vipw -s` | `/etc/shadow` | `/etc/shadow.lock` | `/etc/shadow.OLD` |
| `vigr` | `/etc/group` | `/etc/group.lock` | `/etc/group.OLD` |
| `vigr -s` | `/etc/gshadow` | `/etc/gshadow.lock` | `/etc/gshadow.OLD` |
| `visudo` | `/etc/sudoers` and `/etc/sudoers.d/*` | `/etc/sudoers.tmp` | parse-check before save |
| `vipw -p` | passwd-PAM era; alias on some distros | | |
| `pwck` | Consistency-check `/etc/passwd` + `/etc/shadow` | — | — |
| `grpck` | Consistency-check `/etc/group` + `/etc/gshadow` | — | — |
| `pwconv` / `pwunconv` | Convert shadow ↔ no-shadow | — | — |
| `grpconv` / `grpunconv` | Same for group ↔ gshadow | — | — |

### Field layout cheat sheet

**`/etc/passwd` (7 fields, `:`-separated)**

```
kelvin:x:1000:1000:Kelvin Tobias:/home/kelvin:/bin/bash
   │   │  │    │     │             │           │
   │   │  │    │     │             │           └─ shell
   │   │  │    │     │             └─ home directory
   │   │  │    │     └─ GECOS (full name, comments)
   │   │  │    └─ primary GID
   │   │  └─ UID
   │   └─ "x" → password lives in /etc/shadow
   └─ username
```

**`/etc/shadow` (9 fields)**

```
kelvin:$6$abc...:19884:0:99999:7:::
   │      │       │    │  │   │
   │      │       │    │  │   └─ inactive (days after expiry before account disabled)
   │      │       │    │  └─ warn-before-expiry days
   │      │       │    └─ min days between changes
   │      │       └─ last password change (days since epoch)
   │      └─ hashed password ($6$ = SHA-512)
   └─ username
```
Fields 6–9: max days, warn, inactive, expire (absolute), reserved.

**`/etc/group` (4 fields)**

```
wheel:x:10:kelvin,ec2-user
  │   │ │   │
  │   │ │   └─ comma-separated member list
  │   │ └─ GID
  │   └─ "x" → password lives in /etc/gshadow
  └─ group name
```

**`/etc/gshadow` (4 fields)**

```
wheel:!::kelvin,ec2-user
  │   │ │  │
  │   │ │  └─ members
  │   │ └─ group administrators
  │   └─ "!" = no password set
  └─ group name
```

> **The point of the family tree:** Memorize "7 / 9 / 4 / 4." If a field count is wrong on save, `pwck`/`grpck` will tell you.

---

## 🔬 The Anatomy of a `vipw` Session — In One Diagram

```
$ sudo vipw
┌─────────────────────────────────────────────────────────────┐
│ root:x:0:0:root:/root:/bin/bash                              │
│ ...                                                          │
│ kelvin:x:1000:1000:Kelvin Tobias:/home/kelvin:/bin/bash█     │  ← cursor in vi
│ ~                                                            │
└─────────────────────────────────────────────────────────────┘
   /etc/passwd.lock   exists while vipw is running.
   On :wq             vipw renames /etc/passwd → /etc/passwd.OLD
                      writes the new file in place
                      releases the lock

If a syntax problem is detected:
   "vipw: warning: /etc/passwd file contained bogus lines"
   You'll be asked to re-edit before commit.
```

> **Reading rule:** While `vipw` is running, `useradd`, `usermod`, `passwd`, etc. wait. If you walk away mid-edit, lock those tools out for everyone else. **Save and quit promptly.**

---

## 📚 Safe-Edit Reference Table

| Task | Command | Notes |
|---|---|---|
| Edit `/etc/passwd` | `sudo vipw` | Lock + edit + parse |
| Edit `/etc/shadow` | `sudo vipw -s` | Same for shadow |
| Edit `/etc/group` | `sudo vigr` | |
| Edit `/etc/gshadow` | `sudo vigr -s` | |
| Pick editor | `sudo EDITOR=vi vipw` | vipw inherits `$EDITOR` |
| Edit `/etc/sudoers` | `sudo visudo` | Parse-check syntax |
| Drop-in sudoers file | `sudo visudo -f /etc/sudoers.d/devops` | |
| Validate `/etc/passwd` + `/etc/shadow` | `sudo pwck` | Reports missing/dup entries |
| Validate `/etc/group` + `/etc/gshadow` | `sudo grpck` | |
| Read-only check passwd | `sudo pwck -r` | Report only, no fix |
| Restore prior file | `sudo cp -a /etc/passwd.OLD /etc/passwd` | Use cautiously |
| Migrate to shadow | `sudo pwconv` | Already done on RHEL |
| Demote from shadow | `sudo pwunconv` | Rarely needed |
| Inspect lock | `ls -l /etc/{passwd,shadow,group,gshadow}.lock` | Stale lock → reboot or `rm` after checking no editor is open |
| Compare against backup | `sudo diff -u /etc/passwd.OLD /etc/passwd` | |

> **Rule one of safe edits:** Always run `pwck` and `grpck` after manual edits. Always confirm `diff -u FILE.OLD FILE` shows only your intended change.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Manual `/etc/passwd` and `/etc/shadow` repair questions appear; `vipw`/`vipw -s` is the *correct* tool. |
| **RHCE candidate** | `ansible.builtin.user` wraps these; understanding the underlying database makes failures readable. |
| **SRE / Platform** | When a co-worker breaks `/etc/shadow`, you fix it from a rescue shell + `vipw -s`. |
| **DevOps** | Service-account rotation tools touch shadow; locked editing protects rollbacks. |
| **AI / MLOps** | Shared training clusters have hand-curated `/etc/group` membership for GPU access; `vigr` keeps it consistent. |

---

## 🔧 The 6 Tasks

> Six exam-realistic phases that build **lock-aware editing → audit → rollback** muscle memory.

---

### Task 1 — Set up a test user and inspect databases

```bash
sudo -i

# Create a throwaway user just for this lab
useradd -m -s /bin/bash vipwtest
echo 'vipwtest:Vip!w-Lab123' | chpasswd

# Verify entries exist
getent passwd vipwtest
getent shadow vipwtest
getent group  vipwtest

# Take pre-lab backups
mkdir -p /root/vipw-backups
cp -a /etc/passwd  /root/vipw-backups/passwd.bak
cp -a /etc/shadow  /root/vipw-backups/shadow.bak
cp -a /etc/group   /root/vipw-backups/group.bak
cp -a /etc/gshadow /root/vipw-backups/gshadow.bak
```

---

### Task 2 — `vipw` to change `vipwtest`'s shell

```bash
sudo EDITOR=vi vipw
# Inside vi:
#   /vipwtest        find the line
#   $                end of line
#   F:               jump back to the LAST ':'
#   l                step right onto the shell field
#   C                change to end of line
#   /sbin/nologin    type new shell
#   Esc :wq

# Verify
getent passwd vipwtest
diff -u /root/vipw-backups/passwd.bak /etc/passwd | head -n 20

# Inspect that vipw left a backup
ls -l /etc/passwd /etc/passwd.OLD 2>/dev/null
```

---

### Task 3 — `vipw -s` for password aging

Without using `chage`, edit `/etc/shadow` directly to set `vipwtest`'s **max-days** to 30 and **warn** to 7.

```bash
# Reference format (9 fields, ':'-separated):
# user:hash:lastchg:min:max:warn:inactive:expire:reserved

sudo EDITOR=vi vipw -s
# Inside vi:
#   /vipwtest        find the line
#   :s/^\([^:]*:[^:]*:[^:]*:[^:]*\):[^:]*:[^:]*:\(.*\)$/\1:30:7:\2/
#   (Note: this replaces the 5th and 6th colon-fields with 30 and 7.
#    For most rows the simpler approach is: navigate manually with w and r.)
#   Easier hand-edit:
#     5w to step five colon-words to the right (or just type the new values inline)
#     Replace the 5th field (max) with 30 and the 6th (warn) with 7.
#   :wq

# Verify with chage (read-only)
chage -l vipwtest
diff -u /root/vipw-backups/shadow.bak /etc/shadow | head -n 20
```

---

### Task 4 — `vigr` to add `vipwtest` to `wheel`

```bash
sudo EDITOR=vi vigr
# Inside vi:
#   /^wheel:                  go to the wheel group line
#   A                         append at end
#   ,vipwtest                 add to the comma-separated members list
#                             (if members list was empty, add: vipwtest  — no leading comma)
#   Esc :wq

# Mirror /etc/gshadow members list to match
sudo EDITOR=vi vigr -s
# /^wheel:   then append ,vipwtest to the LAST field
# :wq

# Verify
getent group wheel
id vipwtest
diff -u /root/vipw-backups/group.bak /etc/group | head
diff -u /root/vipw-backups/gshadow.bak /etc/gshadow | head
```

---

### Task 5 — Consistency checks with `pwck` and `grpck`

```bash
sudo pwck -r           # read-only audit of /etc/passwd ↔ /etc/shadow
echo "pwck exit: $?"   # 0 = clean

sudo grpck -r          # read-only audit of /etc/group ↔ /etc/gshadow
echo "grpck exit: $?"  # 0 = clean

# Intentionally break: drop the trailing field from vipwtest's passwd line
# (DO NOT DO THIS WITHOUT vipw on real systems — but for the lab:)
sudo cp -a /etc/passwd /etc/passwd.broken-demo
sudo sed -i.bak -E '/^vipwtest:/ s|(/bin/.*$)||' /etc/passwd
sudo pwck -r           # should now report something
# Restore from backup
sudo cp -a /etc/passwd.broken-demo /etc/passwd
sudo rm -f /etc/passwd.broken-demo /etc/passwd.bak
sudo pwck -r           # back to clean
```

---

### Task 6 — Capstone: rollback workflow

**Task statement:** Using only `vipw`, `vipw -s`, `vigr`, and `pwck`/`grpck`, prove that you can:
1. Save the current state of `/etc/{passwd,shadow,group,gshadow}` as `*.pre-cap` backups.
2. Change `vipwtest`'s shell to `/usr/sbin/nologin` via `vipw`.
3. Change `vipwtest`'s max-days to 60 via `vipw -s`.
4. Add `vipwtest` to the `audio` group via `vigr` and `vigr -s`.
5. Verify with `pwck -r` and `grpck -r` that everything is consistent.
6. Roll back to the `*.pre-cap` state using `cp -a` *while holding the editor lock* (i.e., open the file in `vipw` and use `:r! cat /root/.../passwd.pre-cap` to read the backup over the buffer, then `:wq`). Confirm with `diff -u`.

```bash
sudo -i
cp -a /etc/passwd   /root/vipw-backups/passwd.pre-cap
cp -a /etc/shadow   /root/vipw-backups/shadow.pre-cap
cp -a /etc/group    /root/vipw-backups/group.pre-cap
cp -a /etc/gshadow  /root/vipw-backups/gshadow.pre-cap

# Step 2: change shell
EDITOR=vi vipw
#   /vipwtest    →  F:   l   C   /usr/sbin/nologin    Esc :wq

# Step 3: max-days = 60 (field 5)
EDITOR=vi vipw -s
#   /vipwtest    →   navigate to field 5, change to 60   :wq

# Step 4: add to audio group
EDITOR=vi vigr
#   /^audio:    A    ,vipwtest    Esc :wq
EDITOR=vi vigr -s
#   /^audio:    A    ,vipwtest    Esc :wq

# Step 5: audits
pwck -r  && echo "pwck OK"
grpck -r && echo "grpck OK"
chage -l vipwtest
getent group audio

# Step 6: rollback by reading backup over the buffer inside vipw
EDITOR=vi vipw
#   :%d              delete everything in the buffer
#   :r /root/vipw-backups/passwd.pre-cap
#   1               go to line 1 (the buffer starts with a blank line after :r)
#   dd              remove the leading blank
#   :wq

EDITOR=vi vipw -s
#   :%d     :r /root/vipw-backups/shadow.pre-cap     1   dd   :wq

EDITOR=vi vigr
#   :%d     :r /root/vipw-backups/group.pre-cap      1   dd   :wq

EDITOR=vi vigr -s
#   :%d     :r /root/vipw-backups/gshadow.pre-cap    1   dd   :wq

# Confirm full rollback
diff -u /root/vipw-backups/passwd.pre-cap  /etc/passwd
diff -u /root/vipw-backups/shadow.pre-cap  /etc/shadow
diff -u /root/vipw-backups/group.pre-cap   /etc/group
diff -u /root/vipw-backups/gshadow.pre-cap /etc/gshadow
```

**Expected verification output:**

```text
pwck OK
grpck OK
(no diff output — files identical)
```

**Cleanup**

```bash
userdel -r vipwtest
rm -rf /root/vipw-backups
exit
```

---

## 🔍 Safe-Edit Decision Guide

```
What database do you need to edit?
  ├── /etc/passwd                       → sudo vipw
  ├── /etc/shadow                       → sudo vipw -s
  ├── /etc/group                        → sudo vigr
  ├── /etc/gshadow                      → sudo vigr -s
  ├── /etc/sudoers (and sudoers.d)      → sudo visudo (parse-checked)
  ├── /etc/nsswitch.conf                → vim (no lock; risk is lower)
  ├── /etc/pam.d/<service>              → vim (no lock; do read carefully)
  └── Any file already managed by config-mgmt (Ansible, Puppet) → change the source, not the file
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Created `vipwtest` and pre-lab backups
- [ ] 02 `vipw` to change the shell; `diff -u` shows only one line changed
- [ ] 03 `vipw -s` to set max-days/warn; `chage -l` confirms
- [ ] 04 `vigr` + `vigr -s` to add user to `wheel` (or `audio`)
- [ ] 05 `pwck -r` and `grpck -r` both exit 0
- [ ] 06 Capstone — rollback via `vipw` buffer-replace, verified with `diff -u`

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Edited `/etc/passwd` directly with `vim` | Could clash with `useradd`/`passwd` mid-edit | Use `vipw` |
| Stale `/etc/passwd.lock` blocks new edits | "another process is editing..." | `lsof /etc/passwd.lock` then `rm` if truly orphaned |
| Saved a 6-field passwd entry (missing field) | Login broken, `pwck` complains | Restore from `/etc/passwd.OLD` |
| Forgot `:` between fields | Same | Same — backups exist |
| Used GECOS commas separator with literal `:` | Field count wrong | GECOS allows commas, not colons |
| `vipw -s` opened `/etc/passwd` instead | Forgot `-s` | Re-open with `vipw -s` |
| Edited `gshadow` but not `group` | `pwck`/`grpck` warn about mismatch | Use both `vigr` and `vigr -s` |
| Used `chage` AFTER hand-editing shadow | chage rewrites the file | Either tool is fine — just don't run them at the same time |
| Forgot to `userdel -r` test users | Stale home directories | `userdel -r vipwtest` |
| Edited `/etc/sudoers` with vim and broke sudo | Locked out of root | Use `visudo`; recover with `pkexec visudo` or single-user mode |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- When the exam says "change user X's login shell," use `vipw` *or* `usermod -s`. Either is acceptable; `vipw` is the manual proof that you understand the file format.

**RHCE candidate**
- The Ansible `user` module ultimately writes to these databases — knowing them lets you write good `state: present` plus `shell: /usr/sbin/nologin` declarations and audit playbook output.

**SRE / Platform interview**
- "How do you safely change `/etc/passwd`?" → `vipw`; explain locking + the `.OLD` rotation + `pwck`.

**DevOps**
- Service-account rotation: a CI job that runs `usermod`/`passwd` is fine; if you need a manual fix, `vipw` is the right tool.

**AI / MLOps**
- Shared cluster onboarding scripts run `useradd` + `usermod -aG cuda,docker user`. Audit with `getent`, fix anomalies with `vigr`.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 26 — `vi`/`vim` | The editor inside `vipw` |
| Lab 23 — `diff` | Compare before/after edits |
| Lab 24 — `sed` | Sometimes a more scriptable alternative |
| Lab — `visudo` *(later)* | The cousin for `/etc/sudoers` |
| Lab — `useradd`/`usermod` *(later)* | The non-interactive way to do the same edits |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
