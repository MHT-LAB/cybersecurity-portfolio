# Sniffing Cleartext Credentials with tcpdump and Wireshark

**Domain:** Threat and Vulnerability Management
**Tools used:** tcpdump, Wireshark, Kali Linux, vsftpd, WinSCP
**Type:** Hands-on network traffic analysis lab (WGU D483 Security Operations)

## Scenario

I played both sides of a packet-sniffing scenario: capturing raw traffic on
the wire with tcpdump, then digging into it in Wireshark to see exactly what
an attacker sitting on the same network segment could pull out of it. The
setup involved a web login (HTTP) and a file transfer (FTP) between real
endpoints — a domain controller, a Windows client, and a Kali box acting as
both the sniffer and, for the FTP test, the server. The point was to
demonstrate, hands-on, why cleartext protocols are still a real risk: not in
the abstract, but by actually pulling a password and a certificate file back
out of a capture.

## Approach

**Capturing traffic (tcpdump)**

1. Confirmed the capture tooling and interface were actually usable before
   trusting any results — checked the tcpdump binary path and listed
   available interfaces to confirm which one was live.
2. Ran a few throwaway captures first to get a feel for tcpdump's output:
   default hostname/port resolution vs. `-nn` numeric-only output, and a
   protocol-filtered capture (ICMP only) against known ping traffic, so I
   could confirm the filter syntax worked before relying on it for the real
   captures.
3. Ran a targeted capture scoped to port 80 while logging into a web app,
   and a separate full capture during an FTP file transfer initiated from a
   different client — deliberately triggering the traffic myself so I'd
   know exactly what should be in the resulting pcap.

**Analyzing traffic (Wireshark)**

1. Opened the HTTP capture and worked through a few display filters to
   narrow down to the interesting traffic — filtering by source IP, by TCP
   flags to find the actual data-carrying packets, and finally searching
   the payload directly for the hex signatures of HTTP GET/POST requests
   rather than relying on Wireshark's protocol dissector alone.
2. Used Follow → HTTP Stream to reassemble the full request/response
   instead of reading raw packets, which is what actually exposed the
   submitted login payload in plaintext.
3. Opened the FTP capture, filtered to the `ftp` control channel first to
   confirm the credentials were sent in the clear, then switched to the
   `ftp-data` filter to find the channel actually carrying the transferred
   file.
4. Followed the TCP stream for the data channel, switched the stream view
   from ASCII to Raw (since the file is binary, not text), and saved it
   back out to disk, then verified the reconstructed file was intact and
   readable.

## Findings

- The web login POST request was fully reassembled from the HTTP stream and
  contained the submitted email and password in plaintext JSON — nothing
  about HTTP protects credentials in transit.
- The login attempt itself failed server-side ("Invalid email or
  password"), which doesn't matter from a sniffing perspective: the
  attempted credentials were captured either way, valid or not.
- The FTP control channel exposed the full `USER`/`PASS` exchange in clear
  text, with no obfuscation at all.
- The FTP data channel could be reassembled and saved back out as a
  byte-for-byte usable file — in this case a certificate file — just by
  following the TCP stream and switching to raw output. Nothing about FTP
  as a protocol prevents file exfiltration reconstruction from a capture.
- Both protocols failed for the same underlying reason: no transport-layer
  encryption. The vulnerability isn't a bug, it's the protocol design.

## What I'd do differently / lessons learned

- I'd script the hex-signature filtering (`http contains 474554` /
  `504f5354`) into a saved Wireshark filter set rather than typing it out
  each time — useful for repeat triage of similar captures.
- Worth practicing the same workflow against a TLS-wrapped capture (HTTPS,
  FTPS) to confirm firsthand what Follow Stream looks like when it's
  actually encrypted, as a direct before/after comparison.
- In a real environment I'd pair this with a quick network scan or asset
  inventory to identify which services are still running cleartext
  protocols before someone else finds them on the wire.
- Saving extracted files via Follow Stream works fine at this scale, but
  for larger captures the built-in Export Objects feature would be faster
  than manually following each stream.

## Why this matters for the job

This is the exact workflow behind validating a "cleartext credentials in
transit" finding on a pentest or vulnerability assessment — proving it with
an actual captured payload, not just citing that the protocol is
inherently insecure. It's also directly applicable to SOC packet-capture
triage when investigating a suspected data exfiltration or credential
compromise on an internal segment.
