# Directory Traversal & Command Injection: Exploitation and Log-Based Investigation

**Domain:** Threat and Vulnerability Management / Security Operations
**Tools used:** Kali Linux, Firefox ESR, DVWA (Damn Vulnerable Web Application), LAMP server, Linux CLI (`less`, `grep`), Apache2 access.log and error.log (Combined Log Format)
**Type:** Hands-on lab (WGU D483 Security Operations)

## Scenario
This lab covered two related vulnerability classes against DVWA — directory traversal and command injection — from both sides of the fence: first exploiting each as an attacker, then switching hats to investigate the resulting Apache logs as an analyst confirming what happened. The point wasn't just to prove the vulnerabilities exist, but to connect what an exploitation attempt actually looks like on the wire to what it leaves behind in logs, and to understand where the two available log sources (access.log and error.log) each fall short on their own.

## Approach

### Part 1 — Exploiting directory traversal
1. Logged into DVWA at Low security and opened the File Inclusion page, which loads content via a `page` URL parameter.
2. Established a negative baseline first: requesting a nonexistent directory (`blah/include.php`) returned a blank page, giving me something to compare real results against.
3. Confirmed the app resolves `../` as a literal parent-directory operator by requesting `blah/../include.php` and getting the normal page back — proof the input wasn't being sanitized, just concatenated into a path.
4. Compared relative vs. absolute references: a relative `etc/passwd` (no leading slash) returned blank, but an absolute `/etc/passwd` rendered the file. That told me the app placed no restriction on paths outside its own root once you used an absolute reference.
5. Since a single `../etc/passwd` didn't work but the absolute path did, I incremented the `../` count until it resolved, mapping how many directory levels separated the web root from the filesystem root.
6. Requested `/etc/shadow` to test the boundary between "file doesn't exist" and "file exists but isn't readable" — both return a blank page from this app, which is itself a useful (if frustrating) finding.
7. Pulled `/proc/version` and other `/proc` files (`cpuinfo`, `meminfo`, `uptime`, `modules`) to show this bug isn't limited to static config leaks — it also enables live OS/kernel fingerprinting.

### Part 2 — Investigating directory traversal in the logs
1. On the LAMP box, reviewed `/var/log/apache2/access.log` in Combined Log Format, deliberately setting aside the obvious `/vulnerabilities/fi/` path name since real-world traversal wouldn't be so conveniently labeled.
2. Walked the log oldest-to-newest and reconstructed the same attack sequence from Part 1 purely from request lines: the bad-directory guess, the `../` traversal confirmation, the absolute-path read of `/etc/passwd`, an ascending `../` chain (one through six levels) that mapped web-root depth, the `/etc/shadow` attempt, and the `/proc/version` fingerprinting requests using the discovered six-level depth.
3. Checked response status codes critically rather than at face value: the `/etc/shadow` request returned `200`, but that only confirms Apache processed the request, not that the file's contents were actually served — `shadow` is root-only, so the web account almost certainly couldn't read it. Confirming that definitively would need file-level access auditing, which wasn't enabled here.
4. Noted that absolute-path attacks leave no `../` signature at all, so any detection built only around parent-directory sequences would miss them entirely.
5. Cleared both access.log and error.log afterward per the lab's cleanup step (a lab-only convenience — never appropriate on a real system, since it destroys evidence).

### Part 3 — Exploiting command injection
1. On the Command Injection page, submitted a legitimate IP address first to see the intended behavior (a normal four-reply ping) as a baseline.
2. Tested command stacking with a semicolon (`127.0.0.1; ls -la`), using loopback to keep the ping fast — the result returned the ping output plus a directory listing, confirming the field passes input straight to a shell.
3. Confirmed a second separator also worked (`127.0.0.1 && cat index.php`), which matters because a filter blocking only one separator character wouldn't stop this.
4. Found that a leading semicolon alone (no valid IP first) still executed, meaning the field doesn't require well-formed input before running injected commands.
5. Combined command injection with directory traversal (`; ls -la ..`) to show the injected shell isn't sandboxed to the web directory.
6. Ran a full recon bundle in one request (`; whoami; hostname; ip a; pwd; uptime`), confirming the web process runs as `www-data` — capping the blast radius of this account, but still enough to read the web root, its parents, and world-readable OS files.
7. Used a self-chosen delimiter (`echo ...`) between chained `ls` commands at increasing traversal depth to make multi-command output easier to read.
8. Tried a third separator style, a leading pipe (`|cat /etc/passwd`), confirming pipe-based chaining also works and noting that these are host OS accounts, distinct from DVWA's own application-level user database.

### Part 4 — Investigating command injection in the logs
1. Checked the access.log first and hit its limit immediately: it recorded that a `POST /vulnerabilities/exec/ HTTP/1.1` happened and when, but not what was submitted — access logs in this format don't capture POST body content.
2. Pivoted to error.log, where DVWA happens to log submitted form data as `data-HEAP` values, and used the access-log timestamp as an anchor to search it rather than reading the much larger file cold (falling back to an hour:minute-only search when the exact-second match didn't line up).
3. Ruled out the first hit as benign before treating anything as an IoC: `ip=10.1.16.66&Submit=Submit` is just a valid IP submitted to a form built to accept one — the same shape as the malicious entries, but not malicious content.
4. Worked through subsequent entries recognizing shell metacharacters rather than obviously "bad" text, and recognized percent-encoding in the raw log data (`%26` for `&`, `%3B` for `;`, `%2F` for `/`, `%7C` for `|`) — understanding this is there so the log viewer itself doesn't misinterpret a stored metacharacter as an instruction.
5. Noticed every relevant submission ended in `&Submit=Submit` and switched to `grep -n Submit=Submit error.log` to surface every candidate in one pass instead of searching entry-by-entry.
6. Walked the resulting IoCs in order, matching each back to the exact technique used in Part 3: `;` stacking, `&&` stacking, `;` with no valid IP prefix, `;` combined with traversal, the recon bundle, the delimiter-separated multi-level listing, and the final piped `/etc/passwd` read.

## Findings

**Directory traversal**
- DVWA's `page` parameter performs no path sanitization at Low security — it follows both `../` sequences and absolute paths.
- Absolute references bypass the web root entirely, with no traversal characters required, meaning a detection rule keyed only on `../` would miss this class of request.
- `/etc/shadow` returned blank despite existing, because the web account lacks read permission — the app gives no way to distinguish "wrong path" from "right path, denied," which favors blind enumeration by an attacker.
- `/proc/version` and neighboring files were readable, extending this bug from static file disclosure into live system fingerprinting.
- In the logs, the ascending `../` chain from one to six levels is a clear signature of an attacker measuring web-root depth, and a `200` status on a permission-restricted file request is not proof the file was served.

**Command injection**
- The "Enter an IP address" field concatenates user input directly into a shell command with zero validation.
- `;`, `&&`, and `|` all worked as command separators — a filter stripping only one would be incomplete.
- The web process runs as `www-data`, and command injection here directly enabled traversal and file disclosure, making this a high-severity finding rather than an isolated issue.
- In the logs, the access.log alone cannot detect this attack class at all since it doesn't capture POST bodies; the error.log's incidental logging of form data was the only place the actual payloads were visible.
- Percent-encoded metacharacters (`%3B`, `%26`, `%2F`, `%7C`) appearing in a field meant to hold an IP address are themselves a detection signal, independent of what the decoded command turns out to be.

**Combined evidence table**

| Technique | Payload | Log evidence | Source |
|---|---|---|---|
| Traversal — negative baseline | `page=blah/include.php` | Blank page | access.log |
| Traversal — `../` confirmed | `page=blah/../include.php` | Normal page rendered | access.log |
| Traversal — absolute path | `page=/etc/passwd` | File contents shown | access.log |
| Traversal — depth mapping | `page=../../../../../../etc/passwd` | Resolves at 6 levels | access.log |
| Traversal — permission-denied file | `page=/etc/shadow` | Blank, but `200` status | access.log |
| Traversal — OS fingerprinting | `page=.../proc/version` | Kernel/version info shown | access.log |
| Injection — baseline | `ip=10.1.16.66&Submit=Submit` | Normal ping, benign submission | error.log |
| Injection — `;` stacking | `ip=127.0.0.1; ls -la&Submit=Submit` | Ping + directory listing | error.log |
| Injection — `&&` stacking | `127.0.0.1 && cat index.php&Submit=Submit` | Ping + file contents | error.log |
| Injection — traversal combo | `ip=; ls -la ..&Submit=Submit` | Parent directory listing | error.log |
| Injection — recon bundle | `ip=; whoami; hostname; ip a; pwd; uptime&Submit=Submit` | Full system recon | error.log |
| Injection — pipe | `ip=| cat /etc/passwd&Submit=Submit` | OS account list | error.log |

## What I'd do differently / lessons learned
On the exploitation side, I'd script the traversal-depth discovery and the injection recon payloads instead of building them interactively one field at a time — useful for a single lab file, but too slow against an unfamiliar target with limited time. I'd also test encoded and obfuscated variants of both attacks (percent-encoding, double-encoding, Unicode/overlong UTF-8 for traversal; encoded separators or alternate command substitution for injection) to see whether a more hardened version of this app blocks the literal characters but misses their encoded equivalents — a theme that showed up in both vulnerability classes.

On the investigation side, manually anchoring off a single access-log timestamp and hand-searching the error log doesn't scale past a lab-sized log file. In a real environment I'd want: full request-body logging in a structured, searchable format rather than relying on an error log that happens to capture form data as a side effect; standing detection rules for shell metacharacters (and their encoded forms) in fields that should hold structured data like IP addresses; correlation across access.log and error.log by timestamp or, ideally, a shared request/session ID, since even a few seconds of skew was noticeable in this lab and would be a much bigger problem on a busy production server; and file-level access auditing (e.g., `auditd`) so a `200` status on a sensitive file request doesn't remain ambiguous about whether the file was actually returned.

## Why this matters for the job
Both vulnerability classes here are things I'd expect to find and report in a real pentest or code review — any parameter that builds a filesystem path or shell command from user input needs strict validation or, better, needs to avoid building raw commands from input at all. On the detection side, this lab reinforced that knowing *which* log actually contains the evidence for a given attack class (and being willing to distinguish benign-looking submissions from malicious ones by content, not shape) is core SOC analyst work — not just running `grep`, but understanding what to grep for and why.
