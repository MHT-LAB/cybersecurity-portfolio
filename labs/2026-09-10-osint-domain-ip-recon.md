# OSINT Domain & IP Reconnaissance: tracing a suspicious domain and email-header IP through public registries and DNS

**Domain:** Security Operations
**Tools used:** whois, ARIN, IP2Location, AbuseIPDB, nslookup, dig
**Type:** Hands-on lab (WGU D483 Security Operations)

## Scenario

The setup for this lab was a SOC analyst workflow: I had a domain name (`comptia.org`) and an IP address pulled from a suspicious email header (`217.138.207.226`), and the goal was to build out an OSINT profile on both before deciding whether they warranted escalation. This is basically the first step you'd take on any phishing or suspicious-activity ticket — figure out who owns the infrastructure, where it's hosted, whether it has a history, and how it's configured, before you decide it's worth anyone's time to dig further. The lab environment itself had no internet access, so all the lookups had to happen from my local machine, with results carried back conceptually into the "investigation."

## Approach

**Phase 1 — Domain ownership (WHOIS)**
I started with a WHOIS query against `comptia.org` to establish basic ownership and registration history. This told me who the registrar was, how long the domain had been registered, and confirmed most of the registrant PII was redacted by the registrar — expected for a legitimate, established domain. I also noted the nameservers pointed to Cloudflare, which mattered later since it explained why DNS answers can shift over time (Cloudflare handles the zone) and set up the DNS phases of the lab.

**Phase 2 — IP attribution (ARIN + geolocation)**
Next I took the IP from the email header and ran it through ARIN's registration database. That told me the IP falls within a `/24` block registered to M247 Ltd, a European ISP, and that the block is actually managed under RIPE NCC (the European regional registry) rather than ARIN directly — ARIN just federates the query. I cross-checked with IP2Location for a rough geographic read, which landed on Paris, France, consistent with the RIPE registration.

At this point I made a deliberate call not to treat "Paris + M247" as an identity — M247 is a legitimate hosting/ISP provider, so this only tells me the network the traffic *originated from*, not who's behind the keyboard. IP addresses in email headers can also be spoofed, so I treated this as one data point, not a conclusion.

**Phase 3 — Reputation check (AbuseIPDB)**
To see if this IP had a track record, I ran it through AbuseIPDB. It came back with a ~25% abuse confidence score and multiple community-submitted reports spanning brute-force attempts, web app attacks, fraud orders, phishing, email spam, and spoofing. I weighted this appropriately — AbuseIPDB reports are user-submitted and unverified by the platform, so it's corroborating signal rather than proof. Combined with the ISP ownership, the more likely explanation is that this IP has been used opportunistically by different customers/abusers on M247's network over time rather than being a single dedicated malicious host.

**Phase 4 — DNS enumeration, take one (nslookup)**
With the network-layer picture in hand, I switched to enumerating `comptia.org`'s actual DNS configuration using `nslookup` in interactive mode. I deliberately worked from cached to authoritative data to make sure I trusted the results:
1. Checked which resolver I was querying by default.
2. Pulled an A record for `www.comptia.org` — flagged as "non-authoritative," meaning it came from a caching resolver.
3. Queried the SOA record for `comptia.org` to identify the zone's primary authoritative nameserver and its refresh/retry/expire timers.
4. Resolved that nameserver's own IP address, then repointed `nslookup` at it directly and re-ran the same query — this time with no "non-authoritative" caveat, confirming I was now getting answers straight from the source of truth for the zone.
5. From there, enumerated NS records (both authoritative nameservers), MX records (mail routing), and a CNAME lookup for `email.comptia.org`, which resolved to a Cloudflare-generated hex subdomain rather than something human-assigned — a good example of provider-managed infrastructure vs. hand-configured records.

**Phase 5 — DNS enumeration, take two (dig)**
`nslookup`'s interactive mode is fast to use but doesn't produce clean output for a report, so I repeated the same enumeration with `dig`, which is easier to log and script:
- `dig -t SOA comptia.org` to re-confirm the authoritative nameserver and zone metadata in a single scriptable command.
- `dig -t A armando.ns.cloudflare.com` to resolve that nameserver's IP directly.
- Then, querying **that nameserver specifically** with `@162.159.44.225` for A, MX, NS, and CNAME records against `comptia.org` / `email.comptia.org`, mirroring the nslookup phase but guaranteeing authoritative answers on every single query rather than having to manually switch servers partway through.

## Findings

| Category | Result |
|---|---|
| Domain registrar | Network Solutions |
| Domain registered since | 1995 |
| Registrant PII | Redacted for privacy by registrar |
| Domain nameservers | `armando.ns.cloudflare.com`, `jade.ns.cloudflare.net` |
| Domain abuse contact | `abuse@dns.cloudflare.com` (derived from SOA `mail addr`) |
| Suspicious IP | `217.138.207.226` |
| IP netblock / owner | `217.138.207.0/24`, M247 Ltd |
| IP registry | RIPE NCC (queried via ARIN) |
| IP geolocation | Paris, France |
| IP abuse confidence (AbuseIPDB) | ~25%, multiple reports |
| Abuse categories reported | Brute-Force, Web App Attack, Fraud Orders, Phishing, Email Spam, Spoofing |
| Authoritative nameserver IP | `162.159.44.225` |
| Mail alias (CNAME) target | Cloudflare-generated hex subdomain under `pacloudflare.com` |

Taken together: the domain itself checks out as a long-standing, legitimately registered, Cloudflare-fronted domain with no red flags in its own DNS configuration. The IP from the email header is a different story — it's not inherently malicious infrastructure (it belongs to a real ISP), but it has a meaningful abuse history across several categories relevant to email-based attacks (phishing, spam, spoofing). In a real ticket, I'd flag the IP as suspicious-but-not-confirmed, note the spoofing possibility, and recommend correlating with additional headers (SPF/DKIM/DMARC results, Received chain) rather than closing the investigation on IP reputation alone.

## What I'd do differently / lessons learned

- I'd script the whole domain/IP triage as a single pass instead of jumping between five separate tools and browser tabs — something like a lightweight wrapper that hits WHOIS, RDAP (ARIN's modern API), AbuseIPDB, and dig sequentially and dumps it all into one report. That's the kind of thing worth automating for a real SOC where this happens dozens of times a day.
- `nslookup`'s interactive mode was fine for learning the concepts step by step, but I could tell immediately why `dig` is the tool of choice for anything that needs to go into a report — it's one-shot, scriptable, and the output is far easier to paste into a ticket or parse programmatically.
- I'd want more practice reading RDAP output directly (ARIN's newer replacement for WHOIS) since some registries are moving away from classic WHOIS formatting, and that's likely to keep shifting.
- On the abuse-reputation side, I'd like to get more comfortable pulling from a second or third reputation source (e.g., VirusTotal, Talos) to corroborate AbuseIPDB's crowd-sourced data before drawing conclusions, since a single unverified source shouldn't carry a decision on its own.

## Why this matters for the job

This is the exact triage workflow a SOC analyst runs on nearly every phishing or suspicious-activity ticket — pulling ownership, geolocation, and reputation data on domains and IPs, then validating DNS configuration to spot anomalies, all before deciding whether something escalates to a real incident or gets closed as noise.
