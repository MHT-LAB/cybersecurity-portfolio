# Detecting IoCs in Wazuh: Brute Force, Log Tampering, and Privilege Escalation

**Domain:** Security Operations
**Tools used:** Wazuh (SIEM/IDS), Kali Linux, Hydra, SecLists, Windows Event Viewer, auditpol, net.exe
**Type:** Hands-on SIEM detection lab (WGU D483 Security Operations)

## Scenario

I set up a small monitored environment — a Wazuh server watching a Windows
domain controller through an installed agent — and then played the attacker
from a Kali box to see how three different classes of malicious activity
actually show up (or don't) in SIEM telemetry: brute-forcing a login, wiping
event logs to cover tracks, and tampering with accounts/privileges after
gaining a foothold. The goal wasn't just to trigger alerts, it was to
understand what Wazuh's default rules actually catch, what they mislabel,
and what has to be configured ahead of time (like audit policy) for the
detection to even be possible.

## Approach

**Part 1 — Brute force / credential guessing over RDP and SMB**

1. Built a targeted wordlist by seeding a known password into a standard
   SecLists list, then confirmed placement before running anything, so I'd
   know exactly which attempt should succeed.
2. Ran Hydra against the domain controller's RDP service to simulate an
   online dictionary attack, and separately tested SMB share access — first
   with an invalid account to generate a failure, then with a valid admin
   account to generate a success, so I'd have both outcomes to compare in
   the SIEM.
3. Went into the rule definitions on the Wazuh server itself (not just the
   dashboard) to see how the underlying XML rule actually matches the
   Windows Event ID, instead of trusting the alert label at face value.
4. Compared the rule's assigned MITRE ATT&CK technique against what I
   actually did, which is where the mismatch below came from.

**Part 2 — Anti-forensics (log clearing)**

1. Cleared the Security, Application, and System event logs directly on the
   domain controller through Event Viewer, the way an attacker would after
   gaining local access, to see whether the log-clearing action *itself*
   generates a new, detectable event.
2. Used Wazuh's alert groups widget to filter specifically on Windows log
   events and watched for what fired immediately after each log was
   cleared.
3. Compared severity and rule IDs across the three logs to see if Wazuh
   treats all log-clearing equally or weighs the Security log differently.

**Part 3 — Account tampering and privilege escalation**

1. Enabled full success/failure auditing across all policy categories on
   the domain controller first, and verified it took effect — this
   mattered because without it, later steps wouldn't generate events at
   all.
2. Enumerated local accounts, deleted two, and added a low-privileged
   account to the local Administrators group to simulate post-compromise
   privilege escalation.
3. Filtered out high-volume/low-value noise (routine logon success/logoff
   events) in the Wazuh UI so the account-management alerts weren't
   buried, then confirmed each action mapped to the expected Windows Event
   ID and Wazuh rule.
4. Drilled into the raw JSON for the deletion alert to find exactly where
   the affected username lives in the event schema, since the dashboard
   summary alone didn't show it.

## Findings

- Hydra's RDP brute force succeeded on the pre-seeded attempt, and Wazuh
  caught the authentication event under rule `92652` — but labeled it
  `T1550.002` (Pass the Hash), which isn't what actually happened. It was a
  dictionary guessing attack, not a hash replay. The rule is tuned to a
  Windows Event ID pattern, not the underlying attacker technique.
- SMB access attempts were clearly distinguishable: the invalid-account
  mount failure and the valid-account mount success landed as separate,
  correctly differentiated rule IDs (`60122` failure vs. `60106` success).
- Clearing the **Security** log generated a distinct, high-severity alert
  (rule `60137`, mapped to Windows Event ID 1102, "the audit log was
  cleared") — a direct defense-evasion indicator (`T1070.001`). Clearing
  the **Application** and **System** logs generated a different, lower
  severity event (`60106`/Event ID 104). Wazuh clearly treats loss of
  security auditing as a bigger deal than routine log maintenance.
- Every account-tampering action produced its own event once full auditing
  was enabled: audit policy change (`4719`/rule `60112`), account deletion
  (`4726`/rule `60111`), local group membership change (`4738`/`4735`/rule
  `60147`), and privilege escalation via group add (`4732`/rule `60144`).
  The deleted/affected username specifically lives at
  `data.win.eventdata.targetUserName` in the alert JSON.
- None of the Part 3 events would have existed at all without first
  enabling granular auditing — a SIEM is only as good as the OS-level
  logging feeding it.

## What I'd do differently / lessons learned

- I'd write a custom Wazuh rule (or at least a note for a real detection
  engineering backlog) to correct the `T1550.002` mislabel on rule `92652`
  — or split it into separate rules based on the actual authentication
  method, since mislabeled ATT&CK mapping can send a real analyst down the
  wrong investigative path.
- I'd build this out as a proper Sigma rule mapped to the correct
  technique (likely `T1110.001`, password guessing) so the detection logic
  isn't locked to one SIEM's default ruleset.
- Filtering noisy rule IDs out manually in the UI works for a lab, but in
  a real SOC that logic belongs in a saved search, a correlation rule, or
  a dashboard — not something an analyst rebuilds by hand every session.
- I'd want to test what happens if auditing is only *partially* enabled,
  to know exactly which event IDs go dark first — that's a more realistic
  failure mode than "no auditing at all."

## Why this matters for the job

This is close to the actual workflow of triaging a brute-force or
insider-threat alert queue: confirm what the raw event actually says versus
what the alert label claims, check whether logging was even in place to see
the full picture, and know where in the event schema the actionable
identifiers (usernames, log types) live. The rule-mislabeling piece
especially maps to real detection-engineering work — tuning and correcting
noisy or inaccurate default rules is a constant part of running a SIEM at
scale.
