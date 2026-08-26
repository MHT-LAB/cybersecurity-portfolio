# Threat-Feed-Driven Security Automation: Firewall Blocking, Malware Removal, and DNS Sinkholing

**Domain:** Security Operations, Automation and Orchestration
**Tools used:** Kali Linux, iptables, bash, cron, curl, grep (regex), awk,
file hashing (checksum comparison), `find`
**Type:** CompTIA hands-on lab (WGU D483 Security Operations, CySA+ CS0-003)

This lab had three parts, all built around the same idea: take a threat
intelligence feed and turn it into an automated defensive action instead of
a manual, once-a-day chore. Part 1 automates firewall blocking from an IP
reputation feed. Part 2 automates finding and removing known-bad files from
a file-hash feed. Part 3 automates DNS-level blocking of known-malicious
domains.

## Part 1: Automating firewall blocks from an IP threat feed

### Scenario

The goal was to take a threat intelligence feed of known-malicious IP ranges
(the lab used real historical data from FireHOL, a public IP reputation
project) and turn it into firewall rules, without hand-typing a new rule
every time the feed updates.

### Approach

1. **Confirmed the feed was reachable** and pulled it with `curl` to see the
   raw list of malicious IP ranges.
2. **Sanity-checked the firewall syntax manually first.** Added a single
   `iptables -A INPUT -s <CIDR> -j DROP` rule by hand and confirmed it showed
   up with `iptables -S`, before trying to script anything.
3. **Wrote a bash script (`ip_block.sh`)** that:
   - pulls the feed with `curl`
   - uses a regex (`grep -oE`) to pull out just the IP/CIDR values, dropping
     everything else
   - loops over that array and adds an `iptables -A INPUT -s $IP -j DROP`
     rule for each one, printing a confirmation line as it goes
4. Made it executable (`chmod +x`), ran it, and confirmed the rules landed
   with `iptables -S`.
5. **Scheduled it with cron** to run daily at 1 AM (`0 1 * * *`) so the block
   list stays current without manual intervention.
6. **Tested the full daily-update loop**, not just a single run. Simulated
   a new day by updating the feed and re-running the script, then checked
   the rule list again.
7. That test surfaced a real problem: re-running the script re-adds rules
   for IPs that were already blocked, so the `iptables -S` output filled up
   with duplicate `DROP` rules.
8. **Fixed the duplicate-rule problem** by adding one line to the end of the
   script: `iptables-save | awk '!seen[$0]++' | iptables-restore`. This
   exports the entire current ruleset, uses `awk` to keep only the first
   occurrence of each line (`seen` is an array `awk` uses to remember what
   it's already printed), then reloads the de-duplicated set.
9. Re-ran the script and confirmed the duplicates were gone.

There was also a quick knowledge check embedded in the lab: which line would
correctly add *outbound* blocking to the script. The right answer is
`iptables -A OUTPUT -s $IP -j DROP`. `OUTPUT` is the chain for traffic that
originates from the host itself. `FORWARD` is for traffic being routed
*through* the host (relevant on a router/gateway, not this case), and
`EGRESS`/`EXTERNAL` aren't real iptables chains at all.

### Findings

- End-to-end automation works: feed → parse → block → schedule, verified
  by actually letting a second "day" run through the pipeline instead of
  just checking the first pass.
- The de-dupe fix is what makes it safe to run daily indefinitely. Without
  it, the ruleset grows forever even though the underlying block list barely
  changes day to day.

## Part 2: Automating malware detection and removal from a hash feed

### Scenario

A second feed provided known-bad files as filename + cryptographic hash
pairs (framed as early access to zero-day IoCs, ahead of mainstream AV
databases catching up). The task was to scan the system for those files and
remove only the ones that matched *both* the filename and the hash, not
just the filename alone, since a filename match with a different hash could
be a legitimate file that just happens to share a name with something
malicious.

### Approach

1. Pulled the malware hash feed with `curl` to see the filename/hash pairs.
2. **Reviewed the provided removal script with `less` before running it.**
   Worth calling out on its own: a script that deletes files is exactly the
   kind of thing you read line by line before executing, not just run
   because it was handed to you. The script pulls the feed, extracts each
   filename/hash pair, searches the filesystem for files matching the
   filename, and only deletes a match if its computed hash also matches the
   feed's hash for that filename. A filename-only match is left alone.
3. Identified which hashing algorithm the script used to compare files
   against the feed as part of reviewing it. It uses **SHA-256**. That matters:
   MD5 and SHA-1 both have known collision weaknesses (two different files
   can be crafted to produce the same hash), which is exactly the wrong
   property for a script whose whole job is proving two files are
   identical before deleting one of them. SHA-256 doesn't have that
   practical weakness, which is why it's the right choice for integrity
   verification like this.
4. **Narrowed the scan scope for testing.** The script as written searches
   the whole filesystem (`find /`), which is fine for an overnight cron job
   but too slow to iterate on while testing. Edited it with `vim` to scan
   just `/usr/share` instead (`find /usr/share ...`) so test runs stayed
   fast.
5. Made the script executable and ran it against `/usr/share`.
6. **Confirmed the hash check actually discriminates, not just the
   filename check.** The feed intentionally included a file whose name
   matched something on disk but whose hash in the feed was wrong. The
   script found that file by name, computed its real hash, saw it didn't
   match the feed's (incorrect) hash, and correctly left it alone rather
   than deleting it. The two files where both the filename and hash lined
   up were removed.
7. Verified the removal by listing the directory again. The confirmed-bad
   files were gone. The filename-only match was still there, untouched.
8. Scheduled the script with cron to run nightly at 2 AM
   (`0 2 * * * /bin/bash /root/remove-malware.sh`).

### Findings

- Filename-matching alone is not enough for automated removal. A hash
  check is what prevents the automation from deleting a legitimate file
  that happens to share a name with a known-bad one. Seeing that logic
  actually hold up against a deliberately wrong-hash test case made the
  point better than reading about it would have.
- The script's choice of SHA-256 over MD5/SHA-1 isn't incidental. A
  hash used to justify an automated deletion needs to be collision-resistant,
  or the whole verification step is theater.
- Reading a script fully in `less` before making it executable and running
  it is a habit worth keeping any time the script's job is to delete
  things, especially one written by someone else.

## Part 3: Automating DNS blocking (sinkholing) from a malicious-domain feed

### Scenario

A third feed listed fully-qualified domain names (FQDNs) tied to malicious
activity: phishing, ransomware, botnet command-and-control, and DDoS
infrastructure. The task was to stop this machine from resolving any of
those domains to their real IP addresses at all, by rewriting local DNS
resolution rather than filtering traffic after the fact. This technique has
a name in the field: **DNS sinkholing**. It means pointing a known-bad domain
to a harmless address (usually loopback) instead of letting it resolve
normally.

### Approach

1. Pulled the malicious-domain feed with `curl` to see the list of FQDNs.
2. Wrote a script (`block-DNS.sh`) that:
   - appends a separator line and a `#Bad DNS` header to `/etc/hosts`
   - reads the feed one FQDN per line (`while read fqdn`) and appends
     `127.0.0.1 <fqdn>` to `/etc/hosts` for each one. This is the line
     that makes the local system treat the domain as if it points to
     itself instead of the real malicious server
   - de-duplicates `/etc/hosts` at the end with `awk '!x[$0]++' /etc/hosts
     > /tmp/hosts` followed by `mv /tmp/hosts /etc/hosts`. Writing to a
     temp file first and then moving it over the original matters here:
     redirecting `awk`'s output straight back into `/etc/hosts` in one step
     (`awk '...' /etc/hosts > /etc/hosts`) would truncate the file the
     instant the shell opens it for writing, before `awk` finishes reading
     the original contents, and the whole file would be destroyed. The
     temp-file-then-`mv` pattern avoids that.
   - uses two spaces (not one) in the separator line specifically so it
     won't collide with a legitimate single-space blank line already in
     `/etc/hosts`. The de-dupe step matches on exact line content, so a
     one-space separator could have merged with an unrelated existing blank
     line and behaved unpredictably.
3. Made the script executable and ran it, comparing `/etc/hosts` before and
   after to confirm the new `#Bad DNS` block and entries were added
   correctly.
4. Verified the block actually worked by pinging one of the blocked domains.
   Because `/etc/hosts` is checked before a real DNS query goes out, the
   domain resolved straight to `127.0.0.1`. The ping traffic never left
   the machine to reach whatever real (and potentially malicious)
   infrastructure that domain would otherwise point to.
5. Scheduled the script with cron to run daily at 3 AM
   (`0 3 * * * /bin/bash /root/block-DNS.sh`).

### Findings

- `/etc/hosts` entries take priority over normal DNS resolution on Linux,
  which makes this a fast, no-extra-infrastructure way to block a small,
  known list of bad domains on a single host.
- The lab's own closing note is worth keeping: this only protects the one
  machine whose `/etc/hosts` got edited. Protecting a whole network the same
  way would mean pushing this logic to something all clients actually query.
  That means a DNS server-side block (a DNS Response Policy Zone, for
  example) or a network-wide DNS filtering service (Pi-hole, Cisco Umbrella,
  Quad9, and similar), rather than editing a hosts file on every endpoint
  individually.

## What I'd do differently / lessons learned

- **Part 1:** This version never *removes* a firewall rule. If an IP range
  ages off the feed because it's no longer considered malicious, the block
  stays in place forever. A production version would need to diff against
  the previous pull and remove stale entries, not just append new ones. At
  real-world scale, individual `iptables` rules per IP don't hold up either.
  `ipset` (a single set matched by one iptables rule) is the better tool
  for a large, frequently-updated block list.
- **Part 2:** This script only matches on exact filename + hash. It has no
  way to catch the same malicious file saved under a different name, or a
  legitimately-named file that's actually been swapped for something
  malicious with a different filename. A more complete tool would hash
  everything in scope and check the hash against the feed regardless of
  filename, at the cost of a much longer scan.
- **Part 3:** A hosts-file edit only stops plain DNS lookups on this one
  machine. It does nothing against hardcoded IPs, DNS-over-HTTPS resolvers
  that bypass the system resolver, or any other device on the network. It's
  a fine proof of concept and a fine emergency stopgap on one box, not a
  substitute for blocking at the DNS server or firewall layer.
- **All three:** I'd add real logging with timestamps instead of relying on
  terminal output. A script running unattended at 1, 2, or 3 AM needs an
  audit trail, not output nobody's there to see.

## Why this matters for the job

This is a hand-rolled version of what a lot of commercial security tooling
does behind the scenes. An EDR platform, an MSP's RMM/security stack, or a
SOAR playbook subscribing to a threat feed and pushing IOC-based blocking or
removal out automatically. Building the manual version first is what makes
it possible to actually evaluate a vendor's "automated" feed integration
later: knowing where the failure points are (duplicate/stale firewall rules,
filename-only matching that could delete the wrong file, no expiration
logic) is exactly what lets an analyst tell whether an automated response
pipeline is actually working correctly, or just looks like it is, and
where a false positive could cause real damage (deleting a legitimate file
because a filename check wasn't backed by a hash check).
