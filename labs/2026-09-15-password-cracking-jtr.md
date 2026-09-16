# Password Cracking: John the Ripper (Dictionary/Brute Force) and Wireless PSK Cracking

**Domain:** Threat and Vulnerability Management
**Tools used:** John the Ripper (JtR), aircrack-ng, Kali Linux
**Type:** Hands-on lab (Security Operations)

## Scenario

This lab covered offline password attacks from two angles: cracking NTLM password hashes with John the Ripper, and cracking WPA/WPA2 wireless network passwords with aircrack-ng. Working from a Kali VM, I first attacked a set of NTLM hashes pulled from a target system (MS10) two different ways — a dictionary attack, then a brute-force (incremental) attack — to compare how each performs and where their practical limits show up. I then moved to pre-captured wireless traffic and ran dictionary attacks against WPA and WPA2 handshake captures to see how the same underlying weakness (predictable passwords) plays out on wireless networks, where password exchange security has historically been weaker than it should be.

## Approach

**Dictionary attack**

I started by looking at the hash file itself before attacking it, since knowing what I was working with mattered — noting which account corresponded to the default local administrator (RID 500) and which was the default Guest account (RID 501, normally disabled with a blank password). From there I picked a large, English-focused wordlist (the `xato-net-10-million-passwords.txt` list bundled with Kali) and ran John the Ripper against the hash file in NTLM format. This is an offline attack — it works directly against stolen hashes rather than a live login prompt, which is why it isn't affected by account lockout policies and can run at billions of attempts per second instead of being throttled by an authentication service.

The dictionary run finished almost immediately and cracked all 12 of the target passwords, meaning every password in that hash file existed verbatim somewhere in a 10-million-entry English wordlist. I saved the results with `john --show --format=NT ms10-hashes.txt > dict-cracked.txt` so I had a record of what was recovered before moving to the next phase.

**Brute-force attack**

For the second half, I cleared John's cracked-password cache (`~/.john/john.pot`) so I could watch the brute-force method work against the full hash set from scratch, rather than seeing hashes it already had answers for. I ran John in incremental mode, which tries every possible character combination up to a given length using the full 95-character printable ASCII set — no wordlist, no assumptions about what a password might look like.

I let this run for about 5 minutes and watched the live status updates (triggered by pressing spacebar), which show percent complete and an estimated time to completion. Short, simple passwords fell almost immediately; the harder passwords took progressively longer as length and character complexity increased. I stopped after cracking most of the target set rather than letting it run to full completion.

I then tested two length-capped brute-force runs to see how the search space scales:
- Capped at 6 characters — the ETA displayed was in the range of hours.
- Capped at 7 characters — the ETA jumped to weeks.

I didn't let either of these run to completion — there was no practical reason to burn hours or weeks on a lab exercise once the scaling behavior was clear. The lab notes that removing the length cap entirely (8+ characters) makes John stop calculating an ETA altogether, since each additional character multiplies the search space by roughly 95x.

**Wireless PSK cracking**

For the wireless portion, I worked against pre-captured `.cap` files rather than live-capturing traffic myself, since intercepting real wireless traffic without authorization would run afoul of federal wiretap law — a good reminder that "technically possible" and "legally permitted" aren't the same thing, even in a lab context. I used a combined password list (`passwords29.txt`, built from several of the standard SecLists wordlists) and ran `aircrack-ng` against three different capture files in sequence:

1. A WPA capture using basic PSK authentication.
2. A WPA2 capture also using PSK authentication.
3. A WPA2 capture using the EAPoL 4-way handshake, which is the more modern and generally more secure authentication exchange used by WPA2.

Each attack followed the same pattern — point aircrack-ng at the wordlist and the capture file, and let it check the wordlist against the handshake. All three completed in under a minute.

## Findings

**John the Ripper (NTLM hashes)**

- **Dictionary attack: 12/12 passwords cracked**, and the attack completed in seconds. Every password in this hash set was a common, unmodified word or phrase that existed in a public wordlist — meaning none of the accounts had a password an attacker with a moderately good wordlist couldn't recover almost instantly, offline, with no interaction with the live system at all.
- **Brute-force attack: cracked most of the remaining/simpler passwords within about 5 minutes**, confirming that short or low-complexity passwords fail fast even without a wordlist match, purely from exhaustive search.
- **Scaling cost is the real finding here**: going from an uncapped 5-minute run, to a 6-character cap (hours to finish), to a 7-character cap (weeks to finish) shows the brute-force search space growing so fast that password *length* alone — even using simple lowercase letters — is one of the most effective defenses against this attack type. An 8-character-or-longer password pushes the timeline out far enough that JtR doesn't even bother estimating it.
- Together, the two attacks show a clear pattern: weak/common passwords fall to a dictionary attack in seconds regardless of length, and passwords that dodge a dictionary attack but are still short fall to brute force in minutes. The passwords that survive both are the ones that are both long and not based on a real word or common pattern.

**Wireless PSK cracking**

| Capture File | Auth Type | Password Recovered | Crack Time |
|---|---|---|---|
| wpa.cap | WPA, PSK | `biscotte` | Under 1 minute |
| wpa2-psk-linksys.cap | WPA2, PSK | `dictionary` | Under 1 minute |
| wpa2.eapol.cap | WPA2, EAPoL 4-way handshake | `12345678` | Under 1 minute |

All three networks were cracked using nothing more than a dictionary wordlist, including the WPA2 network using the more modern EAPoL handshake — the stronger authentication exchange didn't matter because the underlying password was weak in every case (a plain dictionary word, the literal word "dictionary," and a simple numeric sequence). This confirms the core weakness isn't really about WPA vs. WPA2 or PSK vs. EAPoL — it's that any pre-shared-key scheme is only as strong as the password chosen, since the whole handshake exists to prove someone knows that shared secret, and if the secret is guessable, the underlying crypto strength barely matters.

## What I'd do differently / lessons learned

I'd want to actually graph or log the incremental status updates over time rather than eyeballing the ETA each time I hit spacebar — for a real writeup or presentation on password policy, having concrete numbers (e.g., "6-char alphanumeric: X hours, 7-char: X weeks") would make a much stronger case to non-technical stakeholders than just saying "brute force takes a long time." I'd also want to try a hybrid approach next time — dictionary words with common substitutions and appended numbers/symbols (a rule-based attack) — since that sits in the realistic middle ground between "used a real word" and "fully random string," and it's closer to how a lot of real user passwords are actually constructed.

Letting the length-capped runs go longer wasn't worth it here, but in a real assessment I'd want compute resources (GPU-based cracking, distributed cracking) benchmarked against realistic password policies before concluding a given minimum length is "safe" — CPU-only incremental mode in a lab VM understates what a well-resourced attacker could do.

For the wireless portion, I ran into a snag trying to count the lines in the password list (`wc -l` didn't behave the way I expected) and didn't get a clean number — next time I'd verify a command like that actually returned before moving on, rather than assuming it worked and losing that data point for the writeup. I also didn't try attacking any of these captures without a wordlist match to see how a hybrid or rule-based approach would fare against something like `biscotte`, which isn't in a plain English dictionary the same way `dictionary` or `12345678` would be — that's a gap in a truly complete before-and-after picture of wireless password strength.

## Why this matters for the job

This is the practical justification behind password policy requirements that show up in every compliance framework: minimum length, complexity, and avoiding dictionary words aren't arbitrary — they directly determine whether a stolen hash dump, or a captured wireless handshake, takes seconds or years to crack. A SOC analyst or IAM-focused role needs to be able to explain *why* a 15-character passphrase beats an 8-character complex password to people who set policy, and being able to point to actual crack-time scaling (rather than just citing best practice) makes that argument a lot more convincing. The wireless piece adds a network-specific angle: it's a reminder that upgrading to a stronger protocol (WPA2's EAPoL handshake over plain PSK) doesn't fix a weak-password problem, which matters when advising on wireless network hardening rather than just endpoint or account security.
