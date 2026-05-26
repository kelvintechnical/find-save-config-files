# Lab: Find Config Files and Save the List — `find`, `>`, `tee`

**Series:** linux-ops-mastery — RHCSA Essential Tools & File Operations
**Subjects covered:** Combining `find` predicates with redirection to capture artifacts; `-name '*.conf'`, `-name '*.cfg'`, multiple `-name` with `-o`, `-type f`, `-not -path` for exclusion, `-print0` for null-terminated output, `-exec ls -lh {} +` for inline metadata, the "package owns" cross-check with `rpm -qf`, and the production reflex of "list every config under `/etc`, save it, then audit it"
**Career arcs covered:** RHCSA ("save the list of all `.conf` files in /etc to /root/conf-list.txt"), RHCE (Ansible `find: patterns='*.conf' paths='/etc' recurse=yes`), SRE (compliance inventories), DevOps (config-as-code bootstrap), AI/MLOps (capture every YAML/JSON config that defines an inference service)
**Prerequisite:** Lab 14 (`find`), Lab 01 (`>`)
**Time Estimate:** 25 to 35 minutes
**Difficulty arc:** Task 1 foundation (find -name '*.conf') · 2 multi-extension via `-o` · 3 exclude with `-not -path` · 4 `-print0` for safe scripting · 5 cross-check ownership with `rpm -qf` · 6 RHCSA exam-realistic capstone

---

## Objective

Produce a clean, scriptable list of configuration files under `/etc` (or anywhere). By the end of this lab you can compose multi-extension `find` queries, exclude noisy subtrees, emit null-terminated lists for safe `xargs` consumption, and cross-reference each file against its owning RPM package.

The capstone is an exam-realistic prompt: *"Save every regular file under `/etc/` whose name ends in `.conf`, `.cfg`, `.ini`, or `.yaml`, excluding anything under `/etc/pki/`, to `/root/all-configs.txt` (newline) and `/root/all-configs.txt0` (null). For each path, also produce a third file `/root/all-configs.tsv` containing `path\towner_pkg` (using `rpm -qf`)."*

> **Lab safety note:** Read-only on `/etc`; writes only to `/tmp/conf-lab` and `/root/`.

---

## Concept: A Config List Is Just a Filtered, Saved Stream

You already know `find` produces matches on stdout. Save them with `>`; preview with `tee`; null-terminate with `-print0` for safe handling.

```
                find /etc -type f -name '*.conf'
                        │
                        │  stdout = one line per match
                        ▼
   ┌──────────────────────────────────────────────────┐
   │  > /root/all-configs.txt        (capture)        │
   │  | tee /root/all-configs.txt    (capture+screen) │
   │  -print0 > /root/all-configs.txt0 (null-terminated)│
   └──────────────────────────────────────────────────┘
```

> **Why this matters:** Inventories are the bedrock of compliance, audits, and migration plans. "Make me a list of every config file we ship" is a one-line `find` + `>` away.

---

## 📜 Why "Config Inventory" Is a Common Task — The Story

Linux's "everything is a file" extends to **everything is a config file under `/etc`**. Daemons, kernel features, network stacks, mail, cron, sudo — all configured by plain-text files in well-known places. RPM packages register every config file they own; `rpm -qf PATH` answers "what installed this?" and `rpm -V` answers "has it changed since install?"

The RHCSA exam frequently asks for a directory listing or a config inventory because **knowing your config landscape is half of administering a Linux system.** Before you can secure it, you need to know what's there.

> **The point of the story:** `find` + `>` is the audit primitive. Pair it with `rpm -qf` to know who owns what, and with `wc -l` to know how many.

---

## 👪 The Config Inventory Family

| Need | Tool |
|---|---|
| Walk and filter | `find` (Lab 14) |
| Save the list | `>`, `>>`, `tee` (Lab 01) |
| Null-terminate | `-print0` + `xargs -0` |
| "Which package owns this?" | `rpm -qf PATH` |
| "Has this file changed since install?" | `rpm -V PKG` or `rpm -Vf PATH` |
| Inventory + count | `find ... \| wc -l` |
| Inventory + diff against baseline | `comm -23 current.txt baseline.txt` |

> **The point of the family tree:** `find` provides the list; `rpm` provides the ownership cross-check; `xargs` consumes the list safely.

---

## 🔬 The Anatomy of a Config-Inventory Pipeline — In One Diagram

```
$ find /etc \( -name '*.conf' -o -name '*.cfg' -o -name '*.ini' \) \
       -type f -not -path '/etc/pki/*' -print0 2>/dev/null \
   | tee >(tr '\0' '\n' > /root/all-configs.txt) \
   | xargs -0 -I{} sh -c 'printf "%s\t%s\n" "$1" "$(rpm -qf "$1" 2>/dev/null || echo unowned)"' _ {} \
   > /root/all-configs.tsv

Breakdown:
   1. find ... -print0    → null-terminated list of paths
   2. tee >(...)          → branch the stream:
        – branch A: convert \0 to \n, save plain list
        – branch B: continue as null stream into xargs
   3. xargs -0 -I{} ...   → run a shell per path
   4. printf "%s\t%s\n"    → emit TSV: path \t owning_pkg
   5. > /root/all-configs.tsv
```

> **Reading rule:** Null-terminated streams + `xargs -0` is the only **safe** way to handle paths that might contain spaces, tabs, or newlines.

---

## 📚 Inventory Reference Table

| Task | Command | Notes |
|---|---|---|
| Single extension | `find /etc -type f -name '*.conf'` | |
| Multiple extensions | `find /etc -type f \( -name '*.conf' -o -name '*.cfg' \)` | Group with `\( \)` |
| Exclude subtree | `find /etc -type f -not -path '/etc/pki/*'` | |
| Null-terminated | `... -print0` | Pair with `xargs -0` |
| Save | `... > /root/list.txt` | |
| Save + display | `... \| tee /root/list.txt` | |
| Save and append | `... \| tee -a /root/list.txt` | |
| Cross-check owner | `xargs -d '\n' -I{} rpm -qf {}` | |
| Cross-check owner (safe) | `xargs -0 -I{} rpm -qf {}` | Null-safe |
| Count matches | `... \| wc -l` | |
| Diff against baseline | `diff baseline.txt new.txt` | |
| Find modified files since install | `rpm -Va \| grep '^..5'` | mod-time / size / md5 changed |

> **Rule one of inventories:** Capture every artifact (the list, the count, the owners). Files are cheap; rebuilding them under pressure is not.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | "List every .conf file under /etc and save to /root/file" is a direct exam variant. |
| **RHCE candidate** | Ansible `find: paths=/etc patterns='*.conf' recurse=yes` mirrors this. |
| **SRE / Platform** | Compliance audits: "show me every config not owned by an RPM" — first-class concern. |
| **DevOps** | Configuration-as-code bootstrap: pull every existing config, vendor into Git, manage with Ansible/Puppet. |
| **AI / MLOps** | "Every YAML defining an inference service" — same pipeline, different extensions. |

---

## 🔧 The 6 Tasks

> Six exam-realistic phases that build the **find → filter → capture → cross-check** habit.

---

### Task 1 — Single-extension inventory under `/etc`

**Purpose:** Save every `.conf` file under `/etc` into a plain text file.

```bash
mkdir -p /tmp/conf-lab && cd /tmp/conf-lab

find /etc -type f -name '*.conf' 2>/dev/null \
  | tee conf-list.txt \
  | wc -l

head -n 5 conf-list.txt
```

**Human-Readable Breakdown:** Recursively search `/etc` for regular files ending in `.conf`. Save through `tee` so you see both the list and the count.

**Reading it left to right:** `-type f` filters to regular files (no directories, symlinks). `-name '*.conf'` matches the extension. `2>/dev/null` silences permission errors. `tee` saves while passing through to `wc -l`.

**The story:** The simplest version of the inventory pattern. Once you have the list saved, you can grep, sort, diff, and ship it.

**Expected output:**

```text
185
/etc/dnf/dnf.conf
/etc/yum.conf
/etc/sysctl.conf
/etc/sysctl.d/99-sysctl.conf
/etc/ssh/sshd_config.d/50-redhat.conf
```

**Switches**

| Token | Meaning |
|---|---|
| `-type f` | Regular files only |
| `-name '*.conf'` | Glob match |
| `\| tee FILE` | Save and pass through |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Permission errors | Add `2>/dev/null` |
| Glob expanded by shell | Quote it |
| `tee` truncated existing file | Use `tee -a` for append |

---

### Task 2 — Multi-extension via `-o`

**Purpose:** Match several extensions in one `find` using `-o` (OR), grouped with `\( \)`.

```bash
find /etc -type f \( -name '*.conf' -o -name '*.cfg' -o -name '*.ini' -o -name '*.yaml' \) 2>/dev/null \
  | tee /tmp/conf-lab/multi-ext.txt \
  | wc -l

awk -F. '{print $NF}' /tmp/conf-lab/multi-ext.txt | sort | uniq -c | sort -rn
```

**Human-Readable Breakdown:** Match four extensions. Save the list. Then use `awk` to count by extension to confirm coverage.

**Reading it left to right:** `\( \)` group the OR predicates. `-type f` applies to whichever matches. `awk -F. '{print $NF}'` prints the last dot-separated field (the extension). `sort | uniq -c | sort -rn` produces a histogram.

**The story:** Real config landscapes mix extensions. One find command can capture them all if you remember to group.

**Expected output:**

```text
234
185 conf
 25 cfg
 14 ini
 10 yaml
```

**Switches**

| Token | Meaning |
|---|---|
| `-o` | Logical OR |
| `\( \)` | Group (escape parens) |
| `awk -F. '{print $NF}'` | Last dot-separated field |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Only first extension matched | Group with `\( \)` |
| Histogram wrong | `awk -F.` splits on every dot — multi-dot filenames produce unexpected `$NF` |

---

### Task 3 — Exclude noisy subtrees with `-not -path`

**Purpose:** Skip `/etc/pki/` (lots of cert symlinks and bundle files) using `-not -path '/etc/pki/*'`.

```bash
find /etc -type f -name '*.conf' 2>/dev/null | wc -l
find /etc -type f -name '*.conf' -not -path '/etc/pki/*' 2>/dev/null | wc -l
find /etc -type f -name '*.conf' -not -path '/etc/pki/*' -not -path '/etc/alternatives/*' 2>/dev/null > /tmp/conf-lab/filtered.txt
wc -l /tmp/conf-lab/filtered.txt
```

**Human-Readable Breakdown:** Count all `.conf` files, then exclude `/etc/pki/`, then also exclude `/etc/alternatives/`. The counts drop with each exclusion.

**Reading it left to right:** `-not -path GLOB` is the negation predicate. Stack multiple `-not -path` predicates to exclude multiple subtrees. (Note: `-prune` is more efficient on huge trees but harder to read.)

**The story:** Real `/etc` has subtrees you don't want in the inventory — certificates, alternatives symlinks, locked package state. Exclude them at find time.

**Expected output:**

```text
198
185
182 /tmp/conf-lab/filtered.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `-not -path GLOB` | Exclude paths matching GLOB |
| Multiple `-not -path` | Stack to exclude multiple subtrees |
| `-prune` | More efficient subtree skip (alternative) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `-not -path` ignored | Wrong glob; verify with bare `find /etc/pki` |
| Performance slow | Use `-prune` for very large trees |
| Excluded too much | The glob is a fnmatch pattern, not a regex |

---

### Task 4 — Null-terminated output for safe scripting

**Purpose:** Use `-print0` and `xargs -0` so paths with spaces or unusual characters survive intact.

```bash
find /etc -type f -name '*.conf' 2>/dev/null -print0 > /tmp/conf-lab/conf.txt0
ls -l /tmp/conf-lab/conf.txt0

# Count using null-terminator
tr '\0' '\n' < /tmp/conf-lab/conf.txt0 | wc -l

# Safe per-file operation
xargs -0 -a /tmp/conf-lab/conf.txt0 -I{} echo "checking: {}" | head -n 3

# Bad alternative (unsafe with spaces)
# xargs < /tmp/conf-lab/conf.txt0 echo
```

**Human-Readable Breakdown:** Emit a null-terminated path list. Convert to newlines just for counting. Consume safely with `xargs -0`.

**Reading it left to right:** `-print0` writes paths separated by `\0` instead of `\n`. `xargs -0 -a FILE` reads null-terminated paths from FILE. `tr '\0' '\n'` is a one-shot conversion for human display.

**The story:** In real configs you sometimes see filenames with spaces (rare under `/etc` but common elsewhere). Null-terminated lists are the only safe format.

**Expected output:**

```text
-rw-r--r--. 1 user user 7402 May 26 ... /tmp/conf-lab/conf.txt0
185
checking: /etc/dnf/dnf.conf
checking: /etc/yum.conf
checking: /etc/sysctl.conf
```

**Switches**

| Token | Meaning |
|---|---|
| `-print0` | Null-terminate output |
| `xargs -0` | Read null-terminated input |
| `xargs -a FILE` | Read from FILE instead of stdin |
| `tr '\0' '\n'` | Convert nulls to newlines |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `xargs` errored on backslashes | Add `-0` or `-d '\n'` |
| `tr` did not convert | Verify nulls present with `od -c \| head` |
| Count mismatch | A path contained a null character — vanishingly rare |

---

### Task 5 — Cross-check ownership with `rpm -qf`

**Purpose:** For each path, ask RPM which package owns it.

```bash
head -n 5 /tmp/conf-lab/conf-list.txt | while read -r f; do
  pkg=$(rpm -qf "$f" 2>/dev/null || echo 'unowned')
  printf "%s\t%s\n" "$f" "$pkg"
done

# Faster batch form
xargs -d '\n' -a /tmp/conf-lab/conf-list.txt rpm -qf 2>/dev/null \
  | paste /tmp/conf-lab/conf-list.txt - \
  | head -n 5
```

**Human-Readable Breakdown:** Loop over the first five paths, ask `rpm -qf` for the owning package, print as TSV. Show a faster `paste`-based form too.

**Reading it left to right:** `rpm -qf PATH` returns the NEVRA of the package owning PATH (or "file not owned by any package"). `paste FILE -` joins lines from FILE with lines from stdin column-wise.

**The story:** Unowned files under `/etc` are usually customizations or leftovers from old packages. Identifying them is a security and hygiene concern.

**Expected output:**

```text
/etc/dnf/dnf.conf        dnf-4.14.0-13.el9.noarch
/etc/yum.conf            dnf-4.14.0-13.el9.noarch
/etc/sysctl.conf         setup-2.13.7-9.el9.noarch
/etc/sysctl.d/99-sysctl.conf  setup-2.13.7-9.el9.noarch
/etc/ssh/sshd_config.d/50-redhat.conf  openssh-server-...
/etc/dnf/dnf.conf  dnf-4.14.0-13.el9.noarch
...
```

**Switches**

| Token | Meaning |
|---|---|
| `rpm -qf PATH` | Owner package |
| `paste FILE -` | Column-join FILE with stdin |
| `while read -r f; do ...; done` | Per-line shell loop |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `not owned by any package` | Customization or non-RPM origin |
| `rpm` not available | Debian-family box; use `dpkg -S PATH` |
| Loop slow on huge lists | Batch with `xargs rpm -qf` instead |

---

### Task 6 — Capstone: RHCSA-realistic three-format inventory

**Task statement:** *"Save every regular file under `/etc/` whose name ends in `.conf`, `.cfg`, `.ini`, or `.yaml`, excluding `/etc/pki/`, to `/root/all-configs.txt` (newline) and `/root/all-configs.txt0` (null). Also produce `/root/all-configs.tsv` containing `path\towner_pkg` (use `rpm -qf`)."*

```bash
sudo -i

find /etc -type f \
  \( -name '*.conf' -o -name '*.cfg' -o -name '*.ini' -o -name '*.yaml' \) \
  -not -path '/etc/pki/*' \
  2>/dev/null \
  | tee /root/all-configs.txt \
  | wc -l

find /etc -type f \
  \( -name '*.conf' -o -name '*.cfg' -o -name '*.ini' -o -name '*.yaml' \) \
  -not -path '/etc/pki/*' \
  -print0 2>/dev/null \
  > /root/all-configs.txt0

# Build TSV
while IFS= read -r f; do
  pkg=$(rpm -qf "$f" 2>/dev/null || echo unowned)
  printf "%s\t%s\n" "$f" "$pkg"
done < /root/all-configs.txt > /root/all-configs.tsv

wc -l /root/all-configs.txt /root/all-configs.tsv
head -n 5 /root/all-configs.tsv

for f in /root/all-configs.txt /root/all-configs.txt0 /root/all-configs.tsv; do
  test -s "$f" && echo "VERIFY: $f exists and is non-empty"
done
```

**Human-Readable Breakdown:** Become root. Produce three artifacts: newline list (`.txt`), null list (`.txt0`), and TSV with package ownership (`.tsv`). Verify all three.

**Layer stack you built:**

```text
/etc
   │
   │  find -type f \( *.conf -o *.cfg -o *.ini -o *.yaml \) -not -path '/etc/pki/*'
   │
   ├── /root/all-configs.txt    (newline-separated paths)
   ├── /root/all-configs.txt0   (null-separated paths)
   └── /root/all-configs.tsv    (path\tpkg)
```

**Expected verification output:**

```text
234
234 /root/all-configs.txt
234 /root/all-configs.tsv
/etc/dnf/dnf.conf       dnf-4.14.0-13.el9.noarch
/etc/yum.conf           dnf-4.14.0-13.el9.noarch
/etc/sysctl.conf        setup-2.13.7-9.el9.noarch
...
VERIFY: /root/all-configs.txt exists and is non-empty
VERIFY: /root/all-configs.txt0 exists and is non-empty
VERIFY: /root/all-configs.tsv exists and is non-empty
```

**Cleanup**

```bash
rm -rf /tmp/conf-lab
rm -f /root/all-configs.txt /root/all-configs.txt0 /root/all-configs.tsv
exit
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| TSV has many `unowned` | Customizations; check `rpm -V` |
| Counts disagree | Path with embedded `\n`; use `-print0` form only |
| Missing `.yaml` files | They may live elsewhere; rerun with broader path |

---

## 🔍 Config-Inventory Decision Guide

```
Need to inventory configs?
  │
  ├── "One extension"
  │       └── ✅ find /etc -type f -name '*.conf' > list.txt
  │
  ├── "Multiple extensions"
  │       └── ✅ find /etc -type f \( -name '*.conf' -o -name '*.cfg' \) > list.txt
  │
  ├── "Exclude a noisy subtree"
  │       └── ✅ find ... -not -path '/etc/pki/*'
  │
  ├── "Need null-terminated for safety"
  │       └── ✅ find ... -print0 > list.txt0
  │
  ├── "Need ownership too"
  │       └── ✅ rpm -qf each path (loop or xargs)
  │
  └── "Find unowned files"
          └── ✅ rpm -Vf / awk on rpm -qf output for 'not owned by any package'
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Single-extension `.conf` inventory under `/etc`
- [ ] 02 Multi-extension via `-o` and grouping
- [ ] 03 Exclude subtrees with `-not -path`
- [ ] 04 Null-terminate with `-print0` for safe consumption
- [ ] 05 Cross-check ownership with `rpm -qf`
- [ ] 06 Execute the RHCSA capstone — three artifacts (newline / null / TSV)

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| `-name '*.conf'` unquoted | Shell expanded | Quote it |
| Missing `\( \)` around OR | First match only | Group |
| Permission noise | Add `2>/dev/null` |
| Used `>` when needed `tee` | No display | Use `tee` |
| TSV breaks on tabs | Use null/print0 or quote fields |
| Counts disagree | A path has `\n` — use `-print0` |
| `rpm -qf` empty | Path not owned or not under RPM |
| Excluded too much | Glob differs from regex |
| Wrote inventory to wrong path | Use absolute `/root/...` |
| Did not save the count | Save `wc -l` to its own file |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Drill the multi-extension `\( -o \)` group plus `>` capture. It is exam-grader-friendly.

**RHCE candidate**
- Ansible `find: patterns='*.conf,*.cfg' paths='/etc' recurse=yes excludes='*pki*'` is the playbook form.

**SRE / Platform interview**
- "How would you check for any unowned config under `/etc`?" → Inventory with find, run each path through `rpm -qf`, filter "not owned by any package."

**DevOps**
- Pre-Ansibleize a server by capturing every existing config and vendor them into Git.

**AI / MLOps**
- "List every YAML defining an inference service" → swap extension, same pipeline.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 14 — Searching with `find` | The predicate foundation |
| Lab 01 — stdout Redirection | The capture mechanics |
| Lab 16 — Search String and Save Output | The grep equivalent |
| Lab 18 — Locate Documentation Files | Find for docs instead of configs |
| Lab — RPM Verify (`rpm -V`) *(later)* | Detect drift in the configs you inventoried |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
