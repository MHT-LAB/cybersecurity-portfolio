# Identifying Legacy Systems with nmap, Metasploit, and EOL Research

**Domain:** Vulnerability Management, Asset Management
**Tools used:** nmap, Metasploit Framework (smb_version), Wikipedia OS version history pages
**Type:** CompTIA hands-on lab (WGU D483 Security Operations, CySA+ CS0-003)

## Scenario

I'm working as a cybersecurity analyst tasked with finding legacy, end-of-life
systems in a company's server network before they turn into unmanaged risk. A
legacy system here means one no longer supported by its vendor, so it stops
getting patches and only grows more vulnerable over time. The lab walks
through three stages of the same problem: find what's actually on the
network and guess its OS with nmap, refine that guess with a more targeted
tool where nmap's guess was fuzzy, then check each OS version against its
real-world support lifecycle to decide which systems are already legacy.

## Approach

**Part 1: nmap discovery and OS fingerprinting**

1. Ran a ping sweep against the server subnet with `nmap 10.1.16.0/24 -sn`
   to find which hosts were actually up, before spending time on a slower
   scan against addresses that don't exist.
2. Fed the discovered addresses into a fast scan with version and OS
   detection: `nmap <hosts> -F -sV -O`. `-F` limits the scan to the top 100
   ports instead of the full top 1000, `-sV` grabs service/version banners,
   `-O` attempts to fingerprint the OS from how the target's network stack
   behaves.
3. Reviewed the results host by host. Some came back with a single
   confident OS guess (MS10 at 10.1.16.2, 98% Windows Server 2016). Others
   came back with a wide range of options and no way to narrow it down from
   the network stack alone (10.1.16.9 came back as anywhere from Windows 7
   through Windows Server 2008 R2 through Windows 8.1).
4. Noticed that the domain controller (DC10, Windows Server 2019, the
   newest OS on the network) gave nmap the least to work with. It sits
   behind a firewall, so nmap only saw open-port responses and never a
   closed-port response to compare against, which is what OS fingerprinting
   actually needs. The newer, better-firewalled system was the hardest one
   to identify remotely, which is backwards from what you'd want if the
   goal is finding old, poorly maintained systems.

**Part 2: Refining fuzzy results with Metasploit**

5. Opened a second terminal and ran `msfconsole`, then searched for and
   loaded `auxiliary/scanner/smb/smb_version`. SMB is a core Windows OS
   service, so its banner tends to give up a specific OS version directly,
   which made it a good tool to point at the fuzzy nmap results.
6. Set `RHOSTS` to each system in turn and ran the module. It confirmed
   MS10's top nmap guess (Windows Server 2016), then went further than nmap
   could on the fuzzy ones: 10.1.16.9 came back as a specific Windows 7
   Ultimate SP1 instead of a five-way range, and 10.1.16.13 came back as
   Windows 2012 R2 Datacenter.
7. On the Linux and FreeBSD hosts, the module either had nothing useful to
   report or, on the one system with no SMB service running at all, had to
   wait out a connection timeout before giving up.
8. The real lesson here isn't "Metasploit beats nmap." It's that they use
   different techniques. nmap's `-O` fingerprints the raw IP stack, which
   works against any host but only yields probabilities. Metasploit's
   `smb_version` (and its sibling modules like `ssh_version`, `http_version`,
   and SNMP's `snmp_enum`) asks a specific application-layer service to
   identify itself, which gives a much more exact answer but only works if
   that specific service happens to be running and reachable.

**Part 3: Checking EOL/EOS status**

9. Looked up each confirmed OS version against its real support lifecycle,
   using Wikipedia's Windows version history, Linux kernel version history,
   FreeBSD version history, and Debian version history pages as reference,
   since Microsoft in particular doesn't keep one single official page with
   every version's end-of-support date.
10. Compared each system's End of Support (or End of Life, for Linux)
    date against today's date to sort current systems from legacy ones.

## Findings

Final host inventory, OS determination, and support status:

| IP | OS | End of Support / EOL | Status |
|---|---|---|---|
| 10.1.16.1 | Windows Server 2019 | 2029-01-09 | Current |
| 10.1.16.2 | Windows Server 2016 | 2027-01-12 | Current |
| 10.1.16.9 | Windows 7 Ultimate SP1 | 2020-01-14 | Legacy |
| 10.1.16.13 | Windows 2012 R2 Datacenter / Server 2012 | 2023-10-10 | Legacy (as of today) |
| 10.1.16.14 | Linux 2.6.9-2.6.33 | As late as Nov 2011 | Legacy |
| 10.1.16.22 | FreeBSD 11.2 | 31 Oct 2019 | Legacy |
| 10.1.16.242 | Linux 4.15-5.6 | As late as June 2020 | Legacy |
| 10.1.16.254 | Linux 4.15-5.6 | As late as June 2020 | Legacy |
| 10.1.16.66 | Debian 12.2.0 (Kali's base) | TBD at time of writing | Current |

Five of the nine systems on this server network are already past their
vendor support date. That's the real finding: not just "here's a list of
IPs and OS versions," but "this network has a legacy system problem" that
needs to go to whoever owns security risk decisions.

A version range from a scanner (like Linux 2.6.9-2.6.33) doesn't give a
single EOL date, since different point releases in that range went end of
life at different times. In that case the honest move is to report the
full range of possible EOL dates rather than pick one and imply more
precision than the scan actually supports.

## What I'd do differently / lessons learned

I'd want to script the EOL lookup step instead of manually checking four
different Wikipedia pages by hand. A real environment with more than nine
hosts would make that manual process painful fast. There are actual EOL
tracking APIs and tools built for exactly this (endoflife.date is a common
one), and tying a scanner's OS output into something like that
automatically would be a natural next step to build out.

I'd also want to double check any single-source EOL claim against a second
source before reporting it as fact in a real ticket. Wikipedia is a
reasonable starting point, and the lab itself points out it's not
authoritative, but a legacy-system finding that drives a real
remediation decision deserves a second source, ideally the vendor's own
lifecycle page when one exists.

## Why this matters for the job

Legacy system identification is asset management applied directly to
vulnerability management. It's not enough to know what's on a network.
You have to know what's still supported, because an EOL system stops
getting patches for good and just accumulates risk with no way to close
it except replacement or serious compensating controls (segmentation,
tighter access control, deeper logging). Chaining nmap's broad discovery
with a targeted tool like Metasploit's smb_version to nail down the fuzzy
cases, then checking each result against a real support lifecycle, is the
same workflow a vulnerability management or asset management team runs on
a recurring cycle, not a one-time lab exercise.
