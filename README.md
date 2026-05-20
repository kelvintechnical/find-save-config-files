# Find and Save Config Files (RHEL 9)
### RHCSA EX200 Lab | Part of [linux-ops-mastery](https://github.com/kelvintechnical/linux-ops-mastery)
> Use the `find` command to locate all `.conf` files in `/etc` owned by root and save the list to `/root/config_files`.

![RHCSA](https://img.shields.io/badge/RHCSA-EX200-EE0000?style=flat&logo=redhat&logoColor=white)
![Topic](https://img.shields.io/badge/Topic-File_Operations-blue)

---

## 📋 Scenario

On **Node1**, locate all files under `/etc` that end with `.conf` and are owned by `root`. Save the full list of file paths to `/root/config_files`. Verify the output file contains the correct entries.

## 🎯 Requirements

1. Search `/etc` and all subdirectories recursively.
2. Match only files (not directories) ending in `.conf`.
3. Match only files owned by `root`.
4. Save the list of full file paths to `/root/config_files`.
5. Verify the output file contains correct entries.

## ✅ Tasks

- Use `find` with `-name`, `-user`, and `-type f` flags
- Redirect output to `/root/config_files` using `sudo tee`
- Verify with `sudo cat /root/config_files`

---

## 📚 Command Decision Map

| Lab Phrase | Question Being Asked | Tool |
|------------|---------------------|------|
| "Find all files" | What command searches the filesystem? | `find` |
| "In /etc and subdirectories" | Where do I start the search? | `find /etc` |
| "End with .conf" | How do I match a filename pattern? | `-name "*.conf"` |
| "Owned by root" | How do I filter by file owner? | `-user root` |
| "Files only" | How do I exclude directories? | `-type f` |
| "Save the list" | How do I write output to a file? | `sudo tee` |
| "Verify entries" | How do I confirm file contents? | `sudo cat` |

---

## 🧠 Big Concept

`find` is a filesystem crawler. Unlike `grep` which searches *inside* files, `find` searches for files themselves based on properties — name, type, owner, size, age. Think of it like a SQL query against your filesystem.

---

## Step 1 — Find and save the file list

```bash
sudo find /etc -type f -name "*.conf" -user root 2>/dev/null | sudo tee /root/config_files
```

**What each piece does:**

| Piece | Meaning |
|-------|---------|
| `find /etc` | Start crawling here — `/etc` is where Linux stores ALL system config files |
| `-type f` | Files only — `f` = file, `d` = directory, `l` = symlink |
| `-name "*.conf"` | Filename must end in `.conf` — `*` is a wildcard meaning "anything" |
| `-user root` | Only return files owned by root |
| `2>/dev/null` | Silence permission errors — `2` is stderr, `/dev/null` is a black hole |
| `\| sudo tee` | Pipe output into tee, which writes to `/root/config_files` as root |

> ⚠️ **Common mistake:** Writing `find /etc f` instead of `find /etc -type f` — the dash on `-type` is required. Without it, `f` is treated as a second search path and silently ignored.

---

## Step 2 — Verify the output file

```bash
sudo cat /root/config_files | head -20
```

> **Why `head -20`?** The full list is 100+ entries. `head -20` shows just the first 20 lines to confirm the file was written correctly without scrolling through everything.

> **Why `sudo cat`?** `/root/` is root-owned — reading it as `ec2-user` returns `Permission denied` even if you wrote the file yourself.

**Expected output (sample — your list will be longer):**
```
/etc/host.conf
/etc/ld.so.conf
/etc/libaudit.conf
/etc/xattr.conf
/etc/request-key.conf
/etc/fuse.conf
/etc/krb5.conf
/etc/mke2fs.conf
/etc/authselect/user-nsswitch.conf
/etc/authselect/nsswitch.conf
...
```

**Spot check — confirm a known file is in the list:**
```bash
sudo grep "httpd.conf" /root/config_files
# Expected: /etc/httpd/conf/httpd.conf
```

---

## 🧠 Key Concepts

| Concept | What it means |
|---------|--------------|
| `find /path` | Recursively searches from that directory downward |
| `-type f` | Match files only — excludes `d` (dirs), `l` (symlinks) |
| `-name "*.conf"` | Glob pattern — `*` matches anything before `.conf` |
| `-user root` | Filter by owning user — use `-group` to filter by group instead |
| `2>/dev/null` | Redirect stderr to nowhere — suppresses permission error noise |
| `tee` | Writes to file AND prints to terminal; use with sudo for root-owned paths |
| `sudo cat /root/` | Always need sudo to read/write inside `/root/` as a non-root user |

---

## 📂 What These Directories Mean

| Directory in output | What it configures |
|---------------------|-------------------|
| `/etc/ssh/` | SSH server and client settings |
| `/etc/httpd/` | Apache web server config |
| `/etc/systemd/` | Boot, logging, login, and sleep behavior |
| `/etc/dnf/` | Package manager settings and plugins |
| `/etc/security/` | Password rules, login limits, access control |
| `/etc/selinux/` | Security-Enhanced Linux policies |
| `/etc/NetworkManager/` | Network connections |
| `/etc/audit/` | System audit logging |
| `/etc/containers/` | Podman/container registry settings |
| `/etc/chrony.conf` | NTP config — the service from the timezone lab |

> `/etc` is the brain of the OS — every service reads its config from somewhere inside here.

---

## 🔗 How This Connects to Previous Labs

| File in output | Lab it connects to |
|----------------|-------------------|
| `/etc/NetworkManager/NetworkManager.conf` | Lab 01 — Static IP |
| `/etc/dnf/dnf.conf` | Lab 02 — Repo Access |
| `/etc/chrony.conf` | Lab 03 — Timezone & Time Sync |
| `/etc/httpd/conf/httpd.conf` | Lab 04 — Search String & Save Output |

---

## ⚠️ Pitfalls

- **Missing dash on `-type f`** → writing `find /etc f` treats `f` as a path, not a flag — always include the dash
- **Using `>` instead of `tee`** → redirect runs as your user, fails writing to `/root/`
- **`-name "*.conf"` is case-sensitive** → misses `*.CONF` files; use `-iname` to match both cases
- **Skipping `2>/dev/null`** → permission error lines appear mixed into your output file
- **Not using `sudo cat` to verify** → `Permission denied` reading `/root/config_files` as ec2-user

---

## ✅ Lab Checklist

- `find /etc -type f -name "*.conf" -user root` with correct flags ✓
- `2>/dev/null` suppresses permission errors ✓
- Output saved to `/root/config_files` via `sudo tee` ✓
- Verified with `sudo cat /root/config_files | head -20` ✓
- Spot checked with `sudo grep "httpd.conf" /root/config_files` ✓

---

## 🔗 Related Labs

- [Configure a Static IP Address](https://github.com/kelvintechnical/linux-ops-mastery/blob/master/01-system-management/README.md)
- [Configure Repository Access](https://github.com/kelvintechnical/linux-ops-mastery/blob/master/01-system-management/configure-repo-access.md)
- [Configure Timezone and Time Synchronization](https://github.com/kelvintechnical/linux-ops-mastery/blob/master/01-system-management/configure-timezone.md)
- [Search for a String and Save Output](https://github.com/kelvintechnical/linux-ops-mastery/blob/master/01-system-management/search-string-save-output.md)
- [Full RHCSA/RHCE Study Guide →](https://github.com/kelvintechnical/linux-ops-mastery)

---

## 👤 Author

**Kelvin R. Tobias** — [kelvinintech.com](https://kelvinintech.com) | [GitHub](https://github.com/kelvintechnical) | [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
