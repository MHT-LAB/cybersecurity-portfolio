# Manually Executing an Incident Response Playbook for a High-CPU Rogue Process

**Domain:** Incident Response, Security Orchestration and Automation
**Tools used:** Windows Command Prompt (wmic, tasklist, taskkill, certutil, dir, tar), VirusTotal, Netcat, PowerShell, Kali Linux
**Type:** CompTIA hands-on lab (WGU D483 Security Operations, CySA+ CS0-003)

## Scenario

A SIEM flagged a workstation running at nearly 100 percent CPU, which isn't normal for that system. Normally a SOAR platform would catch this and respond automatically, but the SOAR was down after a failed reconfiguration. That meant falling back to a manual response using a pre-built incident response playbook instead of letting automation handle it. The playbook covers eight steps: find the rogue process, kill it, hash the file behind it, check that hash against an online malware database, find out who owns the file, archive it with its hash, move that archive to a quarantine system, and remove the file from the original host. I ran the entire playbook using command-line tools at every step instead of the GUI or third-party alternatives the lab offered, to build real command-line fluency instead of leaning on point-and-click tools.

## Approach

### Part 1: Identifying the Rogue Process

Used the WMIC utility to pull CPU usage by process:

```
wmic path Win32_PerfFormattedData_PerfProc_Process get Name,PercentProcessorTime
```

This system runs on two virtual CPUs, so total available processor time across all processes adds up to 200 percent instead of 100. Scrolling through the unsorted output, one process stood out, consuming almost all of it: `HeavyLoad.exe`.

### Part 2: Terminating the Process

Found the process ID with:

```
tasklist /FI "IMAGENAME eq HeavyLoad.exe"
```

That returned PID 4996. Killed it with:

```
taskkill /PID 4996 /F
```

Then ran the same `tasklist` filter again to confirm it was gone. It returned "INFO: No tasks are running which match the specified criteria," confirming the process was actually dead, not just off the visible list for a moment.

### Part 3: Hashing the File

Located the file with a drive-wide search, then hashed it:

```
certutil -hashfile "c:\Users\HeavyLoad.exe" SHA256 > HeavyLoad-Hash.txt
type HeavyLoad-Hash.txt
```

That returned a SHA256 hash of `705204f80158fbfb9216a19bc2cb25a392ff028ce028745ac16f543b6290ec81`. Hashing the file before doing anything else with it matters for two reasons. It's the piece of evidence that goes into an online malware lookup, and it's proof the file wasn't altered later in the process.

### Part 4: Checking the Hash Against VirusTotal

Searched the SHA256 hash on VirusTotal. The result came back indeterminate, meaning the file isn't recognized as known malware, but that isn't the same as a clean bill of health. It just means nobody has seen this exact file before. A brand-new or custom-built piece of malware would return the exact same indeterminate result. That's the real lesson in this step: a hash-based lookup can only catch what's already been seen and cataloged somewhere, so no match doesn't mean no threat.

### Part 5: Identifying the File Owner

Ran:

```
dir /q c:\Users\HeavyLoad.exe
```

The file was owned by the Guest account. That's worth flagging on its own. A properly hardened environment usually has the Guest account disabled or heavily restricted, so a Guest-owned executable sitting in a user profile folder is a red flag independent of anything the hash lookup did or didn't confirm.

### Part 6: Archiving the File with Its Hash

Bundled the executable and its hash file together into one archive:

```
tar -a -c -f "c:\HeavyLoad.zip" "C:\Users\HeavyLoad.exe" "C:\Users\HeavyLoad-Hash.txt"
```

Keeping the hash file inside the same archive as the executable matters for chain of custody. Anyone who opens that archive later has proof of what the file's hash was at the time it was collected, without needing to trust a separate record kept somewhere else.

### Part 7: Moving the Archive to Quarantine

Sent the archive to a Kali VM acting as the quarantine system, using a raw TCP connection between PowerShell and Netcat instead of a dedicated file transfer tool. Kali opened a listener:

```
nc -l -p 1234 > HeavyLoad.zip
```

PowerShell on the affected system then pushed the archive's raw bytes over a TCP socket to Kali's address on that port. Verified the transfer landed clean on the Kali side with:

```
unzip -t HeavyLoad.zip
```

That confirmed no errors in the compressed data.

The lab also offered WinSCP as a method for this step, which I didn't pick. That's a real tradeoff worth calling out. The Netcat and PowerShell method moves the file in plain text with nothing protecting it in transit. WinSCP would have moved the same archive over SFTP, encrypted the whole way. Netcat is faster to stand up with nothing extra to install, but on a real network segment where an attacker might still be listening, WinSCP is the better call for keeping evidence confidential during transfer.

### Part 8: Removing the File from the Host

Deleted the archive and the original files from the affected system:

```
del c:\HeavyLoad.zip
del c:\Users\HeavyLoad*
```

Confirmed both were gone with `dir`. This uses the plain `del` command, which unlinks the file but doesn't overwrite its data on disk. That means the file's contents are technically still recoverable with the right forensic tool until that disk space gets reused. The lab also offered SDelete, a tool that does a secure multi-pass overwrite instead. Plain `del` was good enough here since a verified copy of the file already existed safely on the quarantine system, but SDelete would be the right call in a real incident if the goal is making sure the malicious file can't be recovered from the victim host at all.

## Findings

- The rogue process was `HeavyLoad.exe`, running at PID 4996 and consuming nearly all available CPU on the affected host.
- Its SHA256 hash (`705204f80158fbfb9216a19bc2cb25a392ff028ce028745ac16f543b6290ec81`) came back unrecognized on VirusTotal, an indeterminate result rather than a confirmed clean bill of health.
- The file was owned by the Guest account, a red flag on its own regardless of what the hash lookup returned.
- The file and its hash were preserved in a verified archive on a separate quarantine system before being wiped from the original host, so the evidence stayed intact even after cleanup.

## What I'd do differently / lessons learned

Running the whole playbook through the command line instead of the GUI options made each step easier to document and repeat exactly, since every command sits right there in a terminal history. That's a habit worth keeping for any future incident work, not just this lab.

Two method choices are worth flagging for myself specifically: the file transfer and the deletion step. Netcat got the archive to quarantine fast, but WinSCP would have done the same job encrypted. Plain `del` removed the file from the host, but SDelete would guarantee it couldn't be forensically recovered afterward. Neither call was wrong for a training lab, but in a real incident I'd default to the more secure option on both unless there was a specific reason not to.

I'd also double check a hash value everywhere it gets used before typing it anywhere else. It's easy to introduce a small transcription error when a hash gets read off a screen or retyped by hand instead of copied straight out of the tool that generated it, and a single wrong character in a 64-character hex string is invisible at a glance but breaks the entire point of the lookup.

## Why this matters for the job

This lab is really about what happens when the automation you're supposed to be able to rely on stops working. A SOAR platform going offline isn't a hypothetical. Updates fail, integrations break, and licenses lapse, and when that happens the response still has to happen on schedule. Having a playbook that spells out exactly which commands to run, in what order, with the reasoning behind each one, is what makes it possible for an analyst to step in and do by hand what automation would normally have handled. That's a direct rehearsal for the kind of on-call incident response work a SOC or MSP security team does when the tooling can't be trusted to carry the job on its own.
