# Windows Privilege Escalation & IoC Hunt: Attacking and Then Detecting a UAC Bypass

**Domain:** Threat and Vulnerability Management / Security Operations
**Tools used:** Kali Linux, Metasploit (msfvenom, msfconsole, meterpreter), Windows Event Viewer, PowerShell (auditpol), File Explorer
**Type:** Hands-on lab (WGU D483 Security Operations)

## Scenario

This lab was split into two halves, and I played both roles. In the first half I acted as the attacker: I stood up a reverse shell from a Windows victim host (MS10) back to a Kali attack box, then used a UAC bypass to escalate from a standard user context to SYSTEM, and left some deliberately obvious tracks (new folders, a modified file, a text file with a message) to represent an attacker's post-exploitation footprint. In the second half I switched to the analyst role and had to reconstruct that entire intrusion — initial access, the escalation, and the post-exploitation activity — using nothing but the file system and the Windows Security event log.

## Approach

**Phase 1 — Initial access and reverse shell.**
I generated a payload and stood up a Metasploit listener on Kali, then triggered execution on the victim side the way a real victim would: running a batch file that downloaded and executed the payload. This gave me an initial meterpreter session running in the context of the logged-in standard user. I confirmed the session with `sysinfo` and `getuid`, then tried `getsystem` directly — it failed, which is expected, since a standard user token doesn't have the local privileges getsystem relies on without a supporting exploit. I marked this point in the filesystem (a folder under C:\HR) so I'd have a clean "before escalation" timestamp to hunt for later.

**Phase 2 — Privilege escalation via UAC bypass.**
Since a direct escalation attempt failed, I needed a second exploit path. I backgrounded the first session rather than closing it, so I'd have a fallback if the escalation attempt broke the connection, then searched Metasploit's module list for UAC bypass options. I selected `bypassuac_comhijack`, matched the payload architecture to the target (x64), and — importantly — set a different listener port than the original session so I wouldn't collide with or kill the existing connection. After configuring LHOST/LPORT/SESSION and running the exploit, I landed a second meterpreter session. `getuid` still showed the original user, but a follow-up `getsystem` succeeded this time, now that the bypass had elevated the underlying token. I marked this second point with another folder, giving me a clear "before" and "after" pair to look for in the logs.

**Phase 3 — Post-exploitation actions.**
With SYSTEM-level access, I navigated to the marked directory and ran a few basic post-exploitation actions an attacker might use to obscure or misdirect an investigation: used `timestomp` to rewrite the MACE timestamps on an existing file (EMPLOYEES.csv) to a bogus date, and created a new text file with a short message to simulate a dropped artifact. I then cleanly tore down the sessions (`exit`, `back`, `sessions -K`) rather than leaving them dangling.

**Phase 4 — Switching to defender/analyst mode.**
Before digging into the logs, I cleared the audit policy purely to keep the exercise's log volume manageable for searching — I noted this is *not* something you'd do in a real investigation; the correct move is to preserve and work from copies of the logs, not touch the live audit configuration.

From there I worked backward from the filesystem artifacts instead of trying to read the entire event log linearly:
- I noted the exact creation timestamps of both marked folders and the dropped text file in File Explorer, and confirmed the tampered timestamp on EMPLOYEES.csv via its Properties dialog.
- Using those timestamps as anchors, I searched the Security log for the folder names directly. This is a much faster path to relevant events than paging through thousands of records — if you know *what* changed and roughly *when*, search for the artifact name first and let the timestamp confirm you found the right event.
- The first marked-folder event (ID 4656, object access) pointed to a process name of a suspicious executable running under the logged-in user's own account — my first solid IoC for initial access.
- I pivoted off that process name and searched for it again, this time landing on a network-connection event (Windows Filtering Platform permitted-connection record) showing an outbound connection to an external IP on a nonstandard port — the actual reverse shell callback.
- I repeated the same pattern for the second marked folder, but this time the account/process context was different: the creating process ran with a NULL SID and an elevated token type, evidence that this second folder was created *after* privilege escalation rather than by the original process.
- Walking forward from that point in the log, I found a second outbound connection (different port than the first) from the same elevated process, then a cluster of Token Right Adjusted (event ID 4703) records for that process — the actual technical evidence of the privilege escalation, showing the token gaining SYSTEM-level privileges. The process ID on those matched the process ID tied to the second marked folder's creation, tying the whole chain together.
- Finally, I searched for the tampered file and the dropped text file by name to locate the access-request events (ReadAttributes/WriteAttributes) around the timestamp change, and the write event tied to the text file's creation.

## Findings

- **Initial access:** A suspicious executable, downloaded and run by the logged-in user, established an outbound connection to an external IP on port 4444 — consistent with a reverse shell payload. IoC: the executable's file path plus the outbound connection event (WFP permit) in the Security log.
- **Privilege escalation:** A second process, `rundll32.exe`, was observed initiating a second outbound connection (different port) shortly after the first. The process token showed a NULL SID and elevated token type, and a cluster of 4703 Token Right Adjusted events for the same process ID confirmed the token was elevated to SYSTEM. This lines up with the behavioral signature of a UAC bypass technique that hijacks a COM object to launch a high-integrity process (rundll32/eventvwr-style abuse) rather than prompting the user.
- **Anti-forensics attempt:** A file's MACE timestamps were rewritten to a date years in the past — a timestomping attempt to throw off timeline-based investigation. The Security log still captured the ReadAttributes/WriteAttributes access event even though it doesn't log the specific values changed, which is itself a useful detection point: timestomping doesn't erase the audit trail of the change happening, only the filesystem metadata.
- **Dropped artifact:** A new file was created in the same directory as the marked folders, at a timestamp consistent with the rest of the intrusion timeline, corroborated by a WriteData event in the Security log.
- Both "marked" folder creation events, both network connection events, and the 4703 privilege token events form a consistent, correlatable timeline entirely from Security log entries — no need for third-party EDR tooling to establish the chain in this scenario.

## What I'd do differently / lessons learned

Searching the event log by known artifact name (a filename, a folder name, a suspected process) rather than scrolling chronologically is the only realistic approach once you're dealing with anything close to real event volumes — this lab reinforced that pattern well. In a real environment I'd want this correlation automated: a SIEM query joining object-access events, WFP connection-permitted events, and 4703 token-adjustment events on process ID would turn what took manual `Find` clicks here into a single alertable detection rule. I'd also want to flag the specific combination of "process with NULL SID + elevated token type immediately following a standard-user process creating a similarly named object" as a UAC-bypass indicator worth its own correlation rule, since it showed up clearly in this log and is a fairly well-known abuse pattern. Finally, clearing audit policy to make searching faster is a habit I want to be careful never to bring into a real investigation — preserving the original log state (or working from a copy) has to come first, even when it's inconvenient.

## Why this matters for the job

This lab is basically a compressed version of what a SOC analyst does after an EDR or SIEM alert fires: given a handful of suspicious filesystem artifacts and a timestamp, trace backward through the Security log to reconstruct the full attack chain — initial access, privilege escalation, and post-exploitation actions — and tie it together with concrete IoCs (process names, ports, event IDs) rather than guesswork.
