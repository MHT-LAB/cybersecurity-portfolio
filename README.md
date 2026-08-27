# Dustin Ramsey: Security Operations Lab Notes

Hands-on writeups from CompTIA CySA+ (CS0-003) lab work completed as part of WGU's
D483 Security Operations course, plus related SOC and detection-engineering
practice (LimaCharlie, Huntress, Sigma, threat hunting).

I write these up as I go. The goal is to show how I actually work through a
detection, hunting, or incident-response scenario, not just that I completed a
checklist item.

## Why this exists

I run Mars Hill Technology (an MSP) and I'm finishing a cybersecurity master's
degree at WGU, with CompTIA Security+ and CySA+ along the way. This repo is
where the hands-on side of that lives: what I actually did, the tools I used,
what I found, and what I'd do differently. It's aimed at SOC analyst,
detection engineering, and MSP security roles.

## How to read this

Each file in [`labs/`](./labs) is one lab or practice session, written in my
own words: objective, approach, tools, findings, and takeaway. None of these
reproduce CompTIA's, WGU's, or any vendor's proprietary lab instructions,
screenshots of question text, or answer keys. See [Guardrails](#guardrails)
below.

## Index

_(updated as labs are added, newest first)_

<!-- LAB-INDEX-START -->
- [Removing Unneeded Applications and Services](./labs/2026-08-27-remove-apps-services.md): uninstalling an unused app and removing the FTP role service on Windows Server through the Remove Roles and Features Wizard.
- [Controlling Name Resolution with the Hosts File](./labs/2026-08-27-hosts-file-name-resolution.md): editing /etc/hosts on Kali to force, block, and redirect resolution of a domain, and testing each state with wget.
- [Removing Unneeded Device Drivers to Harden a Windows System](./labs/2026-08-27-device-driver-management.md): using Windows Device Manager to scan for hardware changes, update a driver, disable/enable a device, and uninstall a device to see when a removal actually sticks.
- [Threat-Feed-Driven Security Automation: Firewall Blocking, Malware Removal, and DNS Sinkholing](./labs/2026-08-26-threat-feed-firewall-automation.md): a three-part lab covering bash, iptables, and cron IP blocking (with a duplicate-rule bug found and fixed mid-lab), SHA-256-verified automated malware removal, and DNS sinkholing via /etc/hosts.
<!-- LAB-INDEX-END -->

## Guardrails

- **No copyrighted vendor content.** CompTIA lab instructions, question text,
  and CertMaster material stay out. Every writeup describes methodology and
  findings in my own words: what the scenario was in plain terms, what I did,
  what I saw, what it means.
- **No client data.** Nothing from Mars Hill Technology client environments.
  No client names, IPs, hostnames, configs, or credentials.
- **No live exploits or unpatched disclosures.**

## Stack / current focus

Currently: CompTIA CySA+ (CS0-003) hands-on labs via WGU D483, LimaCharlie EDR,
Huntress, Sigma detection rules, and Security+ (SY0-701).

## Contact

Dustin Ramsey | [LinkedIn] | Mars Hill Technology
