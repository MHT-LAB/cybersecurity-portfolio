# Setting Linux File Permissions with chmod

**Domain:** System Hardening, Access Control
**Tools used:** Kali Linux, chmod, ls -l
**Type:** CompTIA hands-on lab (WGU D483 Security Operations, CySA+ CS0-003)

## Scenario

I'm working as a cybersecurity analyst reviewing file permissions on a
Kali Linux system. An important file, demofile.sh, had read, write, and
execute access open to everyone on the system, owner, group, and others
alike. The goal was to lock it down to what it actually needs: full
access for the owner, execute-only for the group, and nothing at all for
everyone else.

## Approach

1. Confirmed the working directory was /root with `pwd`.
2. Ran `ls -l` on a practice file and read the permission string it
   showed. The first character marks whether it's a file or a directory,
   then there are three groups of three characters after that: owner,
   group, and others, each one showing read, write, and execute as
   r, w, x, or a dash if that permission isn't granted.
3. Practiced changing permissions with symbolic notation, checking with
   `ls -l` after each change:
   - `chmod u+x testfile.txt` added execute for the owner.
   - `chmod g+w testfile.txt` added write for the group.
   - `chmod go-r,u-x testfile.txt` removed read from the group and
     others, and removed execute from the owner, all in one command by
     chaining changes together with commas.
4. Switched to octal notation, where each of the three permission slots
   (owner, group, others) is a single digit from 0 to 7, built by adding
   read (4), write (2), and execute (1):
   - `chmod 777` set full rwx for owner, group, and others.
   - `chmod 740` set full rwx for the owner, read-only for the group, and
     nothing for others.
   - `chmod 654` set read/write for the owner, read/execute for the
     group, and read-only for others.
   - `chmod 644` set read/write for the owner and read-only for the
     group and others.
5. Worked out the octal value for the actual target state on demofile.sh:
   owner needs full access (4+2+1 = 7), group needs execute only (1),
   others need nothing (0). That's 710.
6. Ran `chmod 710 demofile.sh` and confirmed with `ls -l` that the
   permission string now read `-rwx--x---`.
7. Read through the script with `cat demofile.sh` before running it, then
   ran `./demofile.sh` now that it had execute permission. It turned out
   to be a small script that reads and prints the system date straight
   from the file, not anything more involved.

## Findings

- Symbolic notation (`u`, `g`, `o` with `+` or `-`) is good for making a
  targeted change without needing to know or restate the whole
  permission set, and multiple changes can be chained into one command
  with commas.
- Octal notation sets an absolute permission state in a single command.
  There's no ambiguity about what stays and what changes, unlike a plain
  `+` in symbolic notation, which only adds a permission and never clears
  out something that shouldn't be there.
- The math behind octal is simple once it clicks: read is 4, write is 2,
  execute is 1, and each of the three digits in a value like 710 is just
  those numbers added up for that one group (owner, group, or others).

## What I'd do differently / lessons learned

This was the first time I actually watched a permission change happen
step by step instead of just memorizing an octal number off a chart. For
a real fix where I already know the exact end state I want, like locking
demofile.sh down to owner-full, group-execute-only, others-nothing,
octal is the faster and cleaner choice, since it sets the whole
permission string in one shot instead of layering changes onto whatever
was already there. I'd save symbolic notation for smaller, one-off tweaks
where I want to change just one thing and leave everything else alone.

## Why this matters for the job

Overly permissive file permissions, especially something like 777 on a
script or config file, show up constantly in vulnerability scans and
security audits. Being able to read a permission listing at a glance and
fix it cleanly with a single, correct chmod command instead of guessing
is a basic but real Linux hardening skill for SOC and sysadmin-adjacent
security work, and it's the kind of finding an analyst should be able to
both spot and remediate without having to look up the syntax every time.
