# Passive Vulnerability Scanning and OS Fingerprinting with Wireshark

**Domain:** Security Operations, Vulnerability Management (Passive Scanning / Network Reconnaissance)
**Tools used:** Wireshark, ping, PuTTY (SSH client), Windows Defender Firewall (netsh advfirewall)
**Type:** CompTIA hands-on lab (WGU D483 Security Operations, CySA+ CS0-003)

## Scenario

I'm acting as an analyst using Wireshark on a Kali sniffer host to passively
fingerprint systems on a lab network, a domain controller called DC10 and
Kali itself. Passive scanning only works if traffic actually crosses the
sniffing interface, so since this is an isolated lab network with no real
production traffic, the exercise was to generate ICMP and SSH traffic myself
and then read what that traffic gave away about each system, without
touching either system directly.

## Approach

1. Opened Wireshark on Kali and started a capture on eth0. Fixed the pane
   layout back to the classic stacked view first (Packet List on top, Packet
   Details in the middle, Packet Bytes on the bottom), since the current
   Kali build defaults to a side-by-side layout that's harder to read.
2. From a Kali terminal, ran `ping 10.1.16.1 -c 4` against DC10. Four
   replies came back.
3. On DC10, opened an elevated command prompt and added a Windows Defender
   Firewall rule blocking inbound ICMPv4 echo requests, then ran
   `ping 10.1.16.66` against Kali to generate traffic the other direction,
   this time with DC10 as the one starting the conversation.
4. Still on DC10, opened PuTTY and started an SSH session to Kali at
   10.1.16.66, to get some encrypted traffic into the same capture alongside
   the plaintext ICMP.
5. Back on Kali, ran the exact same `ping 10.1.16.1 -c 4` command again.
   This time it came back with 100% packet loss, since the firewall rule
   added in step 3 was now blocking DC10 from answering.
6. Stopped the Wireshark capture and worked through it with display filters:
   - `icmp and ip.src==10.1.16.66` to isolate the ping requests Kali
     originated, then expanded the ICMP header down to the Data field and
     read the raw bytes in the Packet Bytes pane.
   - `icmp and ip.src==10.1.16.1` to do the same for DC10's outbound ping,
     making sure to select the actual Echo request packet and not the reply
     that shows up first in the filtered list.
   - `icmp` with no source filter, to look at the full conversation pattern
     across all three ping rounds.
   - `ssh` to look at the encrypted side of the capture, starting with the
     first client and server packets, then the key exchange packets, then a
     packet from later in the session.

## Findings

- The ASCII interpretation of an ICMP echo request's payload gives away the
  originating OS. Kali's outbound ping carried a repeating string of
  keyboard symbols and numbers. DC10's outbound ping carried a repeating
  string of the alphabet. That matches the known pattern: Linux and Unix
  systems pad the ping payload with symbols, Windows pads it with letters.
- Payload size backs up the same finding. Kali's ICMP data field was 48
  bytes, DC10's was 32 bytes. A 48-byte payload lines up with Linux/Unix, a
  32-byte payload lines up with Windows, so the two data points confirm
  each other.
- Getting this right depends on correctly identifying who started the
  conversation. The reply packet always mirrors whatever payload the
  original request carried, so reading the reply instead of the request
  would point to the wrong OS. Filtering on the source IP of the request
  side of the conversation is what makes this reliable.
- Because the ICMP payload is fully readable in Wireshark, that confirms
  ICMP itself is unencrypted, plaintext, no special access needed to read
  it off the wire.
- The third ping round told its own story just from the pattern in the
  Packet List. Round one and round two both showed a clean request
  immediately followed by a reply, four times each. Round three showed four
  requests with no replies at all. A string of unanswered requests like
  that, as opposed to nothing captured at all, is the fingerprint of a
  target that's up and filtering ICMP, not a target that's offline.
- The SSH side leaked useful metadata even though the session itself is
  encrypted. The first client packet showed the exact client software and
  version in plain text (PuTTY, protocol SSH-2.0). The first server packet
  did the same for the server side (OpenSSH 9.1p1 on Debian). The key
  exchange packets right after that showed the negotiated cipher and MAC
  in plain text too (AES-256 in CTR mode, HMAC-SHA2-256). Every packet
  after the handshake had no readable Info column and no readable ASCII in
  the Packet Bytes pane, which is what encrypted payload looks like in
  Wireshark: present, but unreadable.

## What I'd do differently / lessons learned

Manually clicking through the Packet List to find the first real Echo
request works fine for a handful of packets, but at any real scale I'd
filter directly on `icmp.type==8` to pull only requests instead of eyeballing
which row is a request versus a reply. I also want to remember that this
kind of passive OS fingerprinting only gets you a broad family, Windows
versus Linux/Unix, not a specific version or build. It's a fast first pass
with zero risk of detection, not a full answer, and it would need to be
paired with something like active OS detection or banner grabbing to get
more specific.

## Why this matters for the job

This is the exact workflow behind watching a SPAN or mirror port feed: no
packets sent to the target, nothing that shows up in a target's own logs,
just reading what's already crossing the wire. Being able to pull OS type,
client and server software versions, and even negotiated encryption
parameters out of a capture, all without credentials or direct access to
either host, is real asset-inventory and network-visibility work. It's also
a quick way to catch something that shouldn't be there, like a plaintext
protocol carrying real data across a network where it's assumed everything
sensitive is encrypted.
