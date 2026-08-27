# Network Asset Discovery and Identification with nmap

**Domain:** Vulnerability Management, Asset Discovery
**Tools used:** Kali Linux, nmap
**Type:** CompTIA hands-on lab (WGU D483 Security Operations, CySA+ CS0-003)

## Scenario

I'm working as a cybersecurity analyst tasked with mapping out what's
actually running across three subnets: a servers subnet, a clients
subnet, and a screened subnet (a network with tighter external access
controls, sitting between the internal network and the outside world).
Nmap was the tool for the job. The lab starts light with a simple ping
sweep and builds up through a full port scan, service identification,
and OS fingerprinting, showing why one technique alone isn't enough for
a real asset inventory.

## Part 1: Ping Sweeps Across Three Subnets

Ran a ping sweep (`-sn`, which disables port scanning and just sends
ICMP echo requests) against each subnet separately, saving each result
to its own file with `-oN`:

```
nmap 10.1.16.0/24 -sn -oN server_assets_pingsweep.nmap
nmap 10.1.24.0/24 -sn -oN client_assets_pingsweep.nmap
nmap 172.16.0.0/24 -sn -oN screened_assets_pingsweep.nmap
```

Each sweep reported back whatever hosts actually answered the ping.

## Part 2: Full SYN Port Scan of All Three Subnets

Instead of retyping the same scan command three times, combined the
subnets into one target list first:

```
echo 10.1.16.0/24 > targets.txt
echo 10.1.24.0/24 >> targets.txt
echo 172.16.0.0/24 >> targets.txt
```

Then ran one SYN scan (`-sS`) against that whole list, limited to the top
100 common ports (`-F`), with output written in all three formats at once
(`-oA`):

```
nmap -iL targets.txt -F -sS -oA assets_fast_port_scan
```

A SYN scan sends a TCP SYN and never finishes the handshake, so it's the
quieter option (also called a half-open or stealth scan) compared to a
full TCP connect scan, while still reliably showing whether a port is
open.

This port scan across all three subnets combined found nine devices
total, more than the ping sweeps had found on their own. That's the real
point of running both: a ping sweep alone misses hosts that don't answer
ICMP, whether that's a firewall or just how the host is configured, and
a port scan can still catch those, since a listening TCP port will
respond to a SYN even when ICMP is blocked.

## Part 3: Reviewing the Output Formats

`-oA` writes three files at once: `.nmap` (normal readable output),
`.gnmap` (a compact, greppable version with everything about one host on
a single line), and `.xml` (structured output meant for a stylesheet or
another tool to read).

Compared `.nmap` output (`cat assets_fast_port_scan.nmap | less`) against
the `.gnmap` version. The `.gnmap` file has far less detail, but that's
the tradeoff for putting each host on one line. Used
`cat assets_fast_port_scan.gnmap | grep open` to pull out just the hosts
with open ports instead of scrolling through the full results.

## Part 4: Service Version Scan

Ran a version scan against the same target list and port range:

```
nmap -iL targets.txt -F -sS -sV -oN service_versions.nmap
```

This adds a VERSION column to the results. Nmap builds it by sending a
small set of protocol-specific probes to each open port and comparing
the response against its own signature database, sometimes called
banner grabbing, service identification, or enumeration.

Across the three subnets, the scan found SSH, web (HTTP/HTTPS), DNS,
email, and LDAP services running. No FTP or NTP turned up.

## Part 5: OS Detection Scan

Ran an OS detection scan against the same targets:

```
nmap -iL targets.txt -F -sS -O -oN asset_OSes.nmap
```

Nmap fingerprints the OS by looking at quirks in how a system's TCP/IP
stack responds, things like initial TTL, window size, and how it handles
unusual packets, then compares that fingerprint against its own
database. Pulled just the OS lines out of the results with
`cat asset_OSes.nmap | grep Running`.

The three subnets turned up a mix of Windows, Linux, and FreeBSD systems.

## Findings

- A ping sweep is fast but not a complete asset inventory on its own.
  Some hosts don't answer ICMP, and the follow-up SYN port scan found
  more devices than the sweeps alone did. That's exactly why a ping
  sweep should be treated as a first pass, not the final answer.
- A SYN scan strikes the right balance for general discovery. It
  reliably shows whether a port is open without completing a full TCP
  session the way a connect scan does.
- The three output formats each have a job. `.nmap` is for a person
  reading through results, `.gnmap` is for quick command-line searching,
  and `.xml` is for feeding into another tool or dashboard.
- Version scanning and OS detection work the same way at a high level:
  send something, compare the response against a signature database. One
  is looking at the application layer, what's actually answering on a
  port, and the other is looking at how the operating system behaves at
  the network stack level.
- Getting a real service and OS inventory (SSH, web, DNS, email, and
  LDAP running on a mix of Windows, Linux, and FreeBSD hosts) out of a
  handful of unauthenticated network scans, with nothing installed on
  any target, is a good demonstration of why nmap is still a first-day
  tool for mapping out an environment nobody's touched before.

## What I'd do differently / lessons learned

The lab kept warning that nmap parameter order matters for scoring, but
it's also just good practice outside a lab. A documented, exact command
is something a teammate, or future me, can run again and get the same
result. I'd keep a running command log for any real asset discovery
work, not just get it right once and move on.

The lab only mentioned UDP scanning in passing, mostly to explain why
it's unreliable: no sessions or flags to key off of, closed ports don't
always send anything back, and getting a real result depends on either a
response or a timeout on every single port. UDP services like DNS and
NTP matter for a complete inventory too, so that's worth practicing as
its own follow-up.

Building `targets.txt` up front instead of retyping IP ranges for every
command is a small habit, but a real one. Any time there's more than one
target, a target list file that every downstream command reads from
beats retyping the same ranges over and over.

## Why this matters for the job

Asset discovery is the front end of everything a SOC or vulnerability
management team does. A service that isn't known can't be flagged in a
scan and can't get patched. This lab is a compressed version of a real
recon pass: find what's alive, find what's listening, find what it's
running, find what OS it's on. That's the same sequence a vulnerability
management program runs before doing anything else, and it maps
directly to the asset inventory requirement that shows up in nearly
every compliance framework. You can't protect what you don't know is
there.
