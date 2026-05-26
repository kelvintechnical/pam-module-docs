# Lab: Read PAM Module Documentation (`pam_securetty.so`)

**Series:** linux-ops-mastery — RHCSA TCP Wrappers & PAM
**Subjects covered:** Linux-PAM documentation layout (`/usr/share/doc/pam*/`), **`man pam_securetty`**, **`pam_securetty.so`** purpose (root TTY allowlist), relationship to **`/etc/securetty`**, `zgrep` / `less` navigation in compressed docs, **authselect** awareness on RHEL 9
**Career arcs covered:** RHCSA (PAM module literacy), RHCE (hardening playbooks), SRE (root login policy), DevOps (golden image audits), AI (reducing hallucinated module args)
**Prerequisite:** Lab 72 — Explore PAM Config Files
**Time Estimate:** 30 to 45 minutes
**Difficulty arc:** Task 1 locate doc dirs · 2–3 read shipped README/txt · 4 man page deep read · 5 map to `/etc/securetty` · 6 capstone + cleanup

---

## Objective

Build the habit: **before** you load a PAM module with arguments, you read **vendor documentation** — both **`man pam_<module>`** and the **`/usr/share/doc/pam-*/`** text files shipped with the `pam` package family. This lab focuses on **`pam_securetty.so`**, the module that enforces which **TTY devices root may use for password-based console login**.

You will correlate man-page statements with on-disk **`/etc/securetty`** contents and with the **`auth`** lines in **`/etc/pam.d/login`**. You will note how **`authselect`** on modern RHEL can regenerate PAM stacks — so edits may need to flow through **`authselect`** profiles rather than fragile direct file hacks (production guidance).

> **Lab safety note:** This lab is **read-only** except Task 6 optional restore lines. Do not remove `pam_securetty` from `login` on production systems without a tested remote-admin path.

---

## Concept: pam_securetty Gates *root* on *physical/virtual consoles*

Non-root users are not the target audience of **`pam_securetty`**. The module consults **`/etc/securetty`**: if the current TTY name is **not** listed, **root password logins** on that console are refused (exact behavior interacts with other modules and `nologin`).

```
login on /dev/tty3 as root
        │
        ▼
pam_securetty reads /etc/securetty
        │
        ├─ tty3 listed? ──YES──► continue stack
        └─ missing? ──► AUTH FAIL for root password path
```

> **Why this matters:** **Break-glass** scenarios assume root can log in on console — deleting every tty from `securetty` can brick local recovery unless you boot single-user with initramfs.

---

## 📜 Why pam_securetty Exists — The Story

Early Unix machines sat in open machine rooms. Anyone who could attach a serial terminal could attempt **`root`**. Sun's PAM ecosystem included modules that expressed **site policy** separately from daemon code. **`pam_securetty`** encoded a simple rule: **root may only authenticate on terminals the administrator explicitly trusts**, listed in **`/etc/securetty`**.

As Linux matured, **SSH root logins** became a separate policy knob (`PermitRootLogin`), and **console access** became rarer in cloud images (serial console + SSM / cloud-init instead). Still, **`pam_securetty`** remains relevant for **bare-metal**, **KVM guests**, and **emergency maintenance** — and RHCSA-style questions still expect you to know the file and module name.

> **The point of the story:** `pam_securetty` is not "SSH security." It is **console hygiene** for the most privileged local account.

---

## 👪 The pam_securetty Family — Who Lives There

| Artifact | Role |
|---|---|
| `/etc/pam.d/login` | Usually `auth required pam_securetty.so` |
| `/etc/securetty` | Allowlist of tty device names |
| `man pam_securetty` | Argument reference (`noconsole` etc.) |
| `/usr/share/doc/pam*/txts/README*` | Narrative notes and caveats |

### Adjacent controls

| Control | Layer |
|---|---|
| `/etc/ssh/sshd_config` PermitRootLogin | SSH-specific |
| `/etc/nologin` | Blocks non-root interactive logins (Lab 76) |
| `authselect` | Regenerates pam stacks from profiles |

> **The point of the family tree:** **Layering** — securetty is one tile, not the whole floor.

---

## 🔬 The Anatomy of `man pam_securetty` — In One Diagram

```
NAME
 pam_securetty - Limit root login to secure ttys

SYNOPSIS
 pam_securetty.so [noconsole]

DESCRIPTION
 ...
```

> **Reading rule:** The **SYNOPSIS** line tells you what may appear after the `.so` in pam.d fourth column.

---

## 📚 PAM Documentation Reference Table

| Task | Command | Notes |
|---|---|---|
| Find doc RPM | `rpm -ql pam | grep share/doc` | Paths vary slightly |
| List txts | `ls /usr/share/doc/pam*/txts 2>/dev/null | head` | Module narratives |
| Read README | `sed -n '1,80p' /usr/share/doc/pam/README` | High-level PAM primer |
| Man module | `man pam_securetty` | Primary reference |
| Search keyword | `zgrep -n securetty /usr/share/doc/pam*/txts/*.gz 2>/dev/null | head` | If compressed |

> **Rule one of module docs:** Trust **`man pam_<module>`** first; README second for philosophy.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Can name module + file pair: `pam_securetty` + `/etc/securetty` |
| **RHCE candidate** | Document why playbooks should not clobber authselect bundles |
| **SRE / Platform** | Post-incident: "who edited securetty?" |
| **DevOps** | CIS benchmarks reference console root restrictions |

---

## 🔧 The 6 Tasks

---

### Task 1 — Locate PAM documentation directories

**Purpose:** Map package contents.

```bash
sudo -i
rpm -qf /etc/pam.d/login
rpm -ql pam | grep -E 'share/doc.*pam' | head -n 20
```

**Human-Readable Breakdown:** Confirm `pam` package provides docs path list.

**Reading it left to right:** rpm query file owner → list doc paths.

**The story:** **rpm -ql** is faster than GUI file managers on servers.

**Expected output:**

```text
pam-1.5.1-8.el9.x86_64
/usr/share/doc/pam/README
/usr/share/doc/pam/html/...
```

**Switches**

| Token | Meaning |
|---|---|
| `rpm -ql` | List all files in package |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Different pam version | Paths still under `/usr/share/doc/pam*` |

---

### Task 2 — Read the shipped PAM README header

**Purpose:** Skim upstream narrative for mental model.

```bash
sed -n '1,120p' /usr/share/doc/pam/README 2>/dev/null || sed -n '1,120p' /usr/share/doc/pam-*/README | head
```

**Human-Readable Breakdown:** First ~120 lines usually explain stack concept.

**Reading it left to right:** `sed` print window.

**The story:** **Context before configuration** — understand design goals.

**Expected output:**

```text
Linux-PAM README
...
```

**Switches**

| Token | Meaning |
|---|---|
| `sed -n '1,120p'` | Line slice |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| File missing | `dnf reinstall -y pam` |

---

### Task 3 — Enumerate `txts/` directory for module-specific notes

**Purpose:** Practice discovering lesser-known narratives.

```bash
DOC=$(rpm -ql pam | grep -E '/usr/share/doc/pam[^/]*/txts$' | head -1)
echo "DOCDIR=$DOC"
ls "$DOC" 2>/dev/null | head -n 20
```

**Human-Readable Breakdown:** Programmatically find `txts` dir, list first entries.

**Reading it left to right:** variable → list.

**The story:** Some sites ship **more** in `txts/` than man pages emphasize.

**Expected output:**

```text
DOCDIR=/usr/share/doc/pam/txts
README.pam_access
...
```

**Switches**

| Token | Meaning |
|---|---|
| `head -1` | First matching path |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| No txts dir | Distro packaging difference — rely on `man` |

---

### Task 4 — Read `man pam_securetty` and extract argument list

**Purpose:** Build exam flashcard from authoritative man page.

```bash
MANWIDTH=90 man pam_securetty | sed -n '1,120p'
```

**Human-Readable Breakdown:** Pretty print man page slice without paging trap in scripts.

**Reading it left to right:** `MANWIDTH` wraps nicely → `sed` window.

**The story:** **Arguments** column in pam.d is where subtle bugs live.

**Expected output:**

```text
PAM_SECURETTY(8) ...
NAME
 pam_securetty - Limit root login to secure ttys
...
```

**Switches**

| Token | Meaning |
|---|---|
| `MANWIDTH` | Controls man formatting width |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Empty man | `dnf install -y man-pages` |

---

### Task 5 — Correlate `/etc/securetty` with `login` pam stack

**Purpose:** Show live configuration linkage.

```bash
grep -n 'pam_securetty' /etc/pam.d/login || grep -n 'pam_securetty' /etc/pam.d/system-auth

echo '--- /etc/securetty (first 20 lines) ---'
sed -n '1,20p' /etc/securetty
```

**Human-Readable Breakdown:** Find module invocation; preview tty list.

**Reading it left to right:** grep pam.d → show securetty head.

**The story:** **Evidence chain**: module line exists + file contains tty names.

**Expected output:**

```text
8:auth [user_unknown=ignore success=ok ignore=ignore default=bad] pam_securetty.so
---
tty1
tty2
...
```

**Switches**

| Token | Meaning |
|---|---|
| `[user_unknown=ignore ...]` | advanced control syntax — see `man pam.conf` |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Line only in authselect profile | `authselect current` to see profile |

---

### Task 6 — Capstone + cleanup

**Purpose:** Write `/root/pam-securetty-notes.txt` summarizing module purpose + files; remove notes.

```bash
cat > /root/pam-securetty-notes.txt <<'EOF'
pam_securetty.so:
- Purpose: restrict root password logins to TTYs listed in /etc/securetty
- Seen in: /etc/pam.d/login (often)
- Docs: man pam_securetty + /usr/share/doc/pam/*
EOF
cat /root/pam-securetty-notes.txt
```

**Cleanup**

```bash
rm -f /root/pam-securetty-notes.txt
exit
```

**Human-Readable Breakdown:** No system files altered in this lab path.

**Reading it left to right:** notes → delete.

**The story:** **Documentation lab** closes with a one-page student summary.

**Expected output:**

```text
pam_securetty.so:
- Purpose: restrict root...
```

**Switches**

| Token | Meaning |
|---|---|
| `rm -f` | Force remove without prompt |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Want to keep notes | omit `rm` |

---

## 🔍 PAM Module Documentation Decision Guide

```
Need to configure pam_foo?
  │
  ├── man pam_foo exists?
  │       └── Read SYNOPSIS + DESCRIPTION + EXAMPLES
  │
  ├── Need vendor narrative?
  │       └── rpm -ql pam | grep README / txts
  │
  ├── Need argument grammar for control flags?
  │       └── man pam.conf
  │
  └── authselect-managed system?
          └── Prefer authselect profile over hand-edit regeneration loss
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 `rpm -ql pam` doc path discovery
- [ ] 02 Read `/usr/share/doc/pam/README` header
- [ ] 03 List `txts/` directory
- [ ] 04 `man pam_securetty` argument awareness
- [ ] 05 Correlate `/etc/pam.d/login` with `/etc/securetty`
- [ ] 06 Write/delete summary notes (no persistent system change)

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Confusing securetty with SSH PermitRootLogin | Wrong layer | Read sshd_config man |
| Editing pam without authselect awareness | Next `authselect apply` overwrites | Use profiles |
| Assuming ttyS0 listed | Serial root blocked | Add `ttyS0` if policy requires |
| Skipping man EXAMPLES | Bad arg shapes | Copy from man, adapt |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Recite: **`pam_securetty` + `/etc/securetty`**.

**RHCE candidate**
- Wrap doc reading in README for your role — future you will thank present you.

**SRE / Platform interview**
- Differentiate **console policy** vs **network policy** vs **account expiry**.

**DevOps**
- Bake doc URLs / `man` section headings into internal wiki templates.

**AI / MLOps**
- Ground answers in `man` output, not forum snippets.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 72 — Explore PAM Config Files | Stack reading foundation |
| Lab 75 — Limit root Access (pam_securetty) | Hands-on securetty edit |
| Lab 76 — /etc/nologin | Adjacent login gate |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
