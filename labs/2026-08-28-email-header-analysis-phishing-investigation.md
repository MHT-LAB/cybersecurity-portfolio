# Building Context Awareness on a Suspicious Email Through Header and OSINT Analysis

**Domain:** Security Operations (Indicator Analysis, Phishing/Email Investigation)
**Tools used:** CentralOps Email Dossier, That'sThem Reverse Email Lookup, Have I Been Pwned, MXToolbox Email Header Analyzer, WHOIS, ARIN, IP2Location
**Type:** CompTIA hands-on lab (WGU D483 Security Operations, CySA+ CS0-003)

## Scenario

An employee inbox received an email claiming to be an official order confirmation
for a Microsoft Defender subscription. The display name read "Windows Defender
Order," but the actual sender address was a personal Gmail account. Acting as
the analyst, my job was to work through a chain of free OSINT tools, starting
with the sender address and ending with the full email header, to build enough
context to make a confident call on whether this was safe to ignore or something
that needed to be escalated, all without ever clicking the link in the message
itself.

## Approach

1. Looked at the email itself before touching any tool. The mismatch was
   obvious right away: a message claiming to be from Microsoft, sent from a
   Gmail address, with the visible sender name spoofed to look official.
2. Pulled the raw header next, before doing outside research, to see what
   else it revealed. Both the From and Reply-To fields pointed back to the
   same Gmail address, and one of the internal Received lines named the
   sending machine as a generic auto-generated Windows hostname, not
   something that identifies an actual person or company.
3. Ran the sender address through CentralOps.net's Email Dossier tool.
   Confidence rating came back as 0, Bad address, meaning the tool could not
   verify the mailbox as currently valid and deliverable. Worth noting this
   is one data point, not a verdict on its own, since the domain (gmail.com)
   is obviously a real mail provider even if this specific mailbox didn't
   check out.
4. Ran the same address through That'sThem's reverse email lookup. Zero
   results, not tied to any known individual in their database.
5. Checked Have I Been Pwned on the same address. No breach history
   returned.
6. Took the full header over to MXToolbox's Email Header Analyzer. This is
   where the header's authentication results needed a closer read. SPF,
   DKIM, and DMARC all technically show as passing, but that's because the
   message really was sent through Google's own mail servers, not spoofed
   at the protocol level. Passing authentication only confirms the message
   matches its claimed sending server, it says nothing about whether the
   account behind it or its content can be trusted.
7. In the Relay Information section, traced the hop-by-hop path the message
   took, from the originating machine through a GoDaddy-hosted mail relay
   chain to Google's receiving servers, along with the timestamp at each
   hop.
8. Pulled the mail server hostname from one of the relay hops and ran a
   WHOIS lookup on it. It came back registered to Wild West Domains (a
   GoDaddy-owned registrar) since 1998, registrant contact listed as Go
   Daddy Operating Company in Arizona. That confirms the message actually
   transited real GoDaddy infrastructure, not something spoofed.
9. Took the IP address closest to the original sender in the relay chain
   to ARIN for a number registration lookup. It's registered to M247 Ltd,
   a legitimate European hosting and ISP provider, as part of a Paris-based
   net range.
10. Ran that same IP through IP2Location for a geolocation check, which
    confirmed Paris, France.
11. Put it together: GoDaddy's mail infrastructure and M247's Paris IP
    range are both real, legitimate services. On their own they don't prove
    anything malicious, a legitimate ISP being involved doesn't mean the
    traffic through it is legitimate too. The header's device hostname
    isn't useful either, since it's just the default name Windows assigns a
    machine during setup if no one bothers to change it.

## Findings

- The strongest red flag never needed a single tool: an official-sounding
  Microsoft subscription notice sent from a personal Gmail account, with
  the sender name spoofed to hide that address behind a fake display name.
  Display name spoofing works specifically because most mail clients show
  the friendly name by default, not the underlying address.
- Passing SPF, DKIM, and DMARC does not mean a message is safe. This one
  passed all three because it was genuinely sent through Gmail's real
  servers. Authentication proves the message matches its claimed sending
  infrastructure, it says nothing about whether the sending account or its
  content is trustworthy. A throwaway or compromised account on a
  legitimate provider can pass every authentication check while still
  being a phishing attempt.
- CentralOps rating the sending mailbox as confidence 0 (Bad address) is
  worth logging as one data point in the chain, not a standalone verdict.
  Combined with everything else, it fits the pattern of a disposable
  account created to run a short campaign rather than a real, maintained
  mailbox.
- Zero hits on That'sThem and no breach history on Have I Been Pwned only
  mean the address isn't linked to a known identity or a known breach.
  Absence of a hit is not evidence the address is safe, it just means those
  two particular databases have nothing on it.
- Every infrastructure lookup, the WHOIS on the GoDaddy mail relay, the
  ARIN registration on the Paris IP block, and the geolocation result, came
  back to real, legitimate organizations. None of that proves malicious
  intent by itself. It describes real infrastructure the message happened
  to route through, and any of it, especially the IP, could represent a
  spoofed or compromised system rather than the actual attacker sitting
  there.
- No single tool in this chain gives a verdict on its own. Confidence comes
  from the pattern across all of them together: a spoofed display name,
  a sending address that doesn't check out as currently valid, technically
  passing authentication that doesn't establish trust, no identity or
  breach history, and international relay infrastructure with no business
  reason to be part of this message.

## What I'd do differently / lessons learned

I want to get faster at reading an SPF/DKIM/DMARC breakdown at a glance
instead of stopping to reason through what "pass" actually means in
context every time, since that's the piece most likely to get misread as
"this is legitimate" by someone newer to this. I also noticed the natural
order this chain builds in: identity and reputation checks first
(CentralOps, That'sThem, Have I Been Pwned) since they only need the
address and run fast, then header and infrastructure analysis (MXToolbox,
WHOIS, ARIN, geolocation) once there's a full header to dig into. Running
them in that order kept the investigation moving instead of jumping around.
If this were a real case and the message body had links, the next step
would be dropping those into VirusTotal or a similar sandboxing service
before making any final call, since header analysis alone doesn't tell you
what a link or attachment actually does.

## Why this matters for the job

This is standard Tier 1/2 phishing triage: build enough context to make a
confident call, ignore, report, or escalate, without ever clicking a live
link or opening an attachment. Being able to walk an email identity and
header investigation in a repeatable order, and explain clearly why no
single result in that chain is conclusive on its own, is the same judgment
call a SOC analyst makes routinely working a phishing queue.
