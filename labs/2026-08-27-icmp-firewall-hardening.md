# Hardening a System by Blocking ICMP with Firewall Rules

**Domain:** Vulnerability Management, System Hardening (Attack Surface Reduction)
**Tools used:** Windows Command Prompt (ping), Windows Defender Firewall with Advanced Security
**Type:** CompTIA hands-on lab (WGU D483 Security Operations, CySA+ CS0-003)

## Scenario

I'm working as a cybersecurity analyst across two Windows systems, a
domain controller (DC10) and a workstation (PC10). The hardening
requirement was to stop these systems from responding to ICMP, in this
case ping requests, since that's a common way to sweep a network and
figure out what's alive before attacking it. The task was to check both
systems for compliance first, then fix the one that wasn't.

## Approach

1. From PC10, ran `ping DC10`. Got four replies back, meaning DC10 was
   answering ICMP requests and was not compliant with the hardening
   requirement.
2. Logged into DC10 and ran `ping PC10` the other direction. Got four
   Request timed out messages instead. PC10 was already blocking inbound
   ICMP, which is why it didn't answer.
3. On DC10, opened Windows Defender Firewall with Advanced Security and
   went to Inbound Rules to find the two rules that control ICMP
   responses: File and Printer Sharing (Echo Request - ICMPv4-In) and
   File and Printer Sharing (Echo Request - ICMPv6-In).
4. Opened each rule's properties and set it to Block the connection,
   instead of just disabling the rule, for both the IPv4 and IPv6
   versions.
5. Switched back to PC10 and ran `ping DC10` again. This time it came back
   with four Request timed out messages, confirming DC10 was no longer
   answering ping requests.

## Findings

- Disabling a rule and blocking a rule are not the same thing. A
  disabled allow rule just gets ignored, so if any other rule out there
  also grants the same access, that traffic still gets through. Setting
  a rule to explicitly block is what actually stops it, since a deny
  always wins over any number of allow rules, no matter how many exist.
- ICMP has separate rule sets for IPv4 and IPv6. Blocking just the
  ICMPv4-In rule and leaving ICMPv6-In alone would leave the system
  answering pings over IPv6 even after IPv4 looked locked down. Both had
  to be set.
- Testing from both directions before making any change was what
  actually caught the problem. PC10 pinging DC10 showed DC10 was out of
  compliance. DC10 pinging PC10 showed what compliant already looked
  like, since PC10 was already blocking it. That gave a clear before and
  after to compare against once DC10 was fixed.

## What I'd do differently / lessons learned

The instinct to just disable the offending rule would have been the wrong
move here, or at least not a safe one, since Windows Defender Firewall
ships with more than one rule that can grant the same kind of access. An
explicit block is the only way to be sure, and it's a habit worth keeping
any time there's a real chance of overlapping allow rules, not just this
one lab. I also want to remember the Export Policy option on Windows
Defender Firewall with Advanced Security going forward. Exporting the
current policy before making a change like this means there's a real way
back if a firewall change breaks something, instead of falling back to
Restore Defaults, which wipes out everything, not just the one change
that needs undoing.

## Why this matters for the job

Blocking ICMP is a real, common hardening control, since letting every
system on a network answer ping requests makes it trivial for an attacker
(or a legitimate but noisy scanning tool) to map out what's alive before
going after it. The bigger lesson generalizes past ICMP though. Any time
a system is still doing something it shouldn't, the first question is
whether an allow rule needs disabling or whether there's a broader allow
somewhere else that makes an explicit deny the only real fix. That's the
same kind of judgment call a SOC analyst makes reviewing firewall rule
sets during an audit or after a finding comes back non-compliant.
