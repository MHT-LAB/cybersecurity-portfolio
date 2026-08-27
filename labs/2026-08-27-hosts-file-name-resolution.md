# Controlling Name Resolution with the Hosts File

**Domain:** System Hardening, Network Name Resolution
**Tools used:** Kali Linux, wget, nano, /etc/hosts
**Type:** CompTIA hands-on lab (WGU D483 Security Operations, CySA+ CS0-003)

## Scenario

I'm working as a cybersecurity analyst on a Kali Linux system, looking at
how the local hosts file controls where a domain name actually points. A
domain called juiceshop.local was already set up to resolve through a
static entry in /etc/hosts instead of normal DNS. The goal was to see what
happens to that resolution when the hosts file entry changes, first to a
broken value, then to a different but still working address, and to
understand why that matters for hardening a system.

## Approach

1. Ran wget on juiceshop.local before changing anything. It resolved
   through the existing hosts file entry and downloaded index.html,
   confirming the baseline worked.
2. Checked /etc/hosts with cat and found the static entry already there.
   That's a normal way private networks pin an internal domain name to a
   specific server without relying on a DNS server for it.
3. Opened /etc/hosts in nano and changed the IP address on the
   juiceshop.local line to 203.0.113.249, an address with nothing
   listening on it.
4. Ran wget again. It resolved to the new, broken address and just hung,
   waiting on a connection that was never going to come through. Had to
   Ctrl+C out of it.
5. Edited /etc/hosts again, this time pointing juiceshop.local at
   203.0.113.228, a different address that does have a working web server
   behind it.
6. Ran wget one more time. It resolved to .228 and pulled down a page
   successfully, saved as index.html.1 since a file named index.html
   already existed from the first run.

## Findings

- The hosts file always wins. It gets checked before a DNS query even
  goes out, so whatever is in there overrides the real DNS answer
  completely, whether it's correct or not.
- The correct syntax is IP address first, then the domain name, separated
  by at least one space or a tab (`203.0.113.228 juiceshop.local`).
  Lining entries up in columns with extra spaces is just for
  readability, it doesn't change how the entry works.
- One file can do three different things depending on what's in it: force
  resolution to a specific known-good address, block a domain by pointing
  it at an address that isn't listening (or at loopback), or redirect a
  domain to a completely different server that is still up and
  responding. That last one is the most dangerous version, since nothing
  about the connection looks broken.

## What I'd do differently / lessons learned

The part that stuck with me is how normal the redirect case looked.
Pointing juiceshop.local at .228 didn't throw an error or hang the way the
broken entry did. It just quietly served a different site and called it
done. If that other server had been made to look enough like the real
juiceshop site, a user wouldn't notice anything wrong from the browser
alone. That's the real risk with a tampered hosts file, it doesn't look
broken, it looks normal. On a real system I'd want file integrity
monitoring on /etc/hosts (or the Windows equivalent) so a change like this
gets flagged the moment it happens instead of found later.

## Why this matters for the job

This is the manual, single-entry version of the same idea behind the DNS
sinkhole script from an earlier lab session: blocking or redirecting a
domain by controlling what it resolves to, instead of filtering traffic
after the fact. Knowing that this file always overrides DNS is also a
real triage skill. If a ticket says a site is loading wrong or sending
someone to the wrong page, checking /etc/hosts (or
C:\Windows\System32\drivers\etc\hosts on Windows) for a tampered entry is
one of the first places to look, and it's easy to miss if you only think
in terms of DNS servers and forget the local file that gets checked
first.
