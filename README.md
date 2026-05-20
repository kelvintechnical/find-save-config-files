
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

- Use `find` with `-name`, `-user`, and `-type` flags
- Redirect output to `/root/config_files` using `tee`
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

## Step 1 — Find and save the file list

```bash
sudo find /etc -type f -name "*.conf" -user root | sudo tee /root/config_files
```

> **What each flag does:**
> - `find /etc` — start search here, recurse all subdirectories automatically
> - `-type f` — files only, excludes directories and symlinks
> - `-name "*.conf"` — filename must end in `.conf` (case-sensitive)
> - `-user root` — owned by root
> - `| sudo tee` — writes to `/root/config_files` (root-owned directory requires sudo on the write)

---

## Step 2 — Verify the output file

```bash
sudo cat /root/config_files
```

**Expected output (sample — your list will be longer):**
```
/etc/httpd/conf/httpd.conf
/etc/resolv.conf
/etc/nsswitch.conf
/etc/ssh/sshd_config
/etc/systemd/system.conf
/etc/dnf/dnf.conf
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
| `tee` | Writes to file AND prints to terminal; handles root-owned paths correctly with sudo |
| `sudo cat /root/` | Always need sudo to read/write inside `/root/` as a non-root user |

---

## ⚠️ Pitfalls

- **Forgetting `-type f`** → includes directories named `*.conf` in results
- **Using `>` instead of `tee`** → redirect runs as your user, fails writing to `/root/`
- **`-name "*.conf"` is case-sensitive** → misses `*.CONF` files; use `-iname` to match both
- **Permission denied lines in output** → some `/etc` subdirs are restricted; add `2>/dev/null` to suppress errors
- **Not using `sudo cat` to verify** → `Permission denied` reading `/root/config_files` as ec2-user

---

## Bonus — Suppress permission errors

```bash
sudo find /etc -type f -name "*.conf" -user root 2>/dev/null | sudo tee /root/config_files
```

> `2>/dev/null` silences `Permission denied` warnings from restricted directories — keeps output clean.

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
```

Ready to run it on the AMI?
