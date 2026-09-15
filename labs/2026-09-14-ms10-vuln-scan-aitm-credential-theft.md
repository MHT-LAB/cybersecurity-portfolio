# MS10 Server Assessment: Nmap Vulnerability Scanning and Adversary-in-the-Middle Credential Theft

**Domain:** Security Operations
**Tools used:** Nmap (NSE `vuln`/`default` script categories), Kali Linux, Burp Suite Community Edition, Apache2, PowerShell, Firefox, OWASP Juice Shop
**Type:** Hands-on lab (WGU D483 Security Operations)

## Scenario

I was assigned to assess a target Windows Server 2016 host (MS10) on a lab network from a Kali attack box, playing both the analyst and the attacker. The first half of the exercise was a straightforward vulnerability scan of MS10 to see what an unauthenticated network scan would reveal about the host — open services, OS fingerprint, and any flagged weaknesses. The second half switched gears into a social-engineering-driven adversary-in-the-middle (AitM) attack: get a victim user on the domain to route their traffic through my Kali box, then intercept and recover their web login credentials in plaintext. The point of both was the same — show how much an attacker can learn or steal without ever needing valid credentials to start.

## Approach

**Part 1 — Vulnerability scanning**
1. From Kali, I ran an nmap scan against the MS10 target combining several capabilities in one pass: the NSE `vuln` script category to check for known vulnerability signatures, the default `-sC` script set, a version/banner-grab scan (`-sV`) to identify what's actually listening on each open port, and OS detection (`-O`). I saved full output to a file rather than just watching it scroll by, since a scan like this can take several minutes and I wanted a record to review afterward instead of relying on scrollback.
2. Once the scan finished, I reviewed the saved output with `less` rather than re-running anything, and used `grep` to pull specific lines (e.g., searching for `OS` to jump straight to the OS-detection results) instead of paging through the whole file for one data point.
3. From the output I confirmed the target was fingerprinted as Windows Server 2016, and that nmap identified a running SMTP service on TCP port 25, with the mail server software's own banner giving away its name and version — a good example of why banner grabbing (`-sV`) matters: it doesn't just say "port 25 is open," it tells you what's actually running there, which is what you need to look up known CVEs for that specific version.

**Part 2 — Adversary-in-the-middle credential theft**
1. I configured Burp Suite as a transparent proxy listener bound to the Kali box's IP on port 8080, with intercept turned off so traffic would pass through and log silently in the HTTP History tab rather than pausing for manual approval on every request — the goal was passive collection, not active tampering.
2. I wrote a small batch script and hosted it on Kali's local Apache web server. The script's only job was to flip two registry values on a Windows victim machine (`ProxyServer` and `ProxyEnable`) so the victim's browser would silently start routing its traffic through my Kali box. This is the actual mechanism of the attack — the "hack" isn't exploiting a software vulnerability, it's convincing a user's own OS to redirect its traffic for me.
3. The social engineering piece was a phishing email impersonating internal IT support, telling the target user their proxy settings needed updating via a linked script, and dangling an unrelated incentive (a free item on an internal retail site) to get them to log in afterward — a classic combination of authority (IT support) and a small reward to lower suspicion.
4. Acting as the victim, I downloaded and ran the script, which silently applied the proxy change — at that point, my Kali box was sitting in the middle of all of that machine's web traffic.
5. I then visited the incentive site as the victim and attempted to log in with the credentials referenced in the phishing email. The login itself failed (wrong account for that particular site), but that didn't matter — the point wasn't a successful login on the target site, it was that the credentials were transmitted in plaintext over HTTP before the failure was ever returned.
6. Back on Kali, I filtered Burp Suite's HTTP History by MIME type to cut down the noise and located the specific failed login POST request. The request body contained the submitted email address and password in cleartext, confirming the credentials had been fully captured in transit.

## Findings

| Area | Result |
|---|---|
| Target OS (nmap `-O`) | Windows Server 2016 |
| Notable open service | TCP 25 (SMTP) — service banner identified the running mail server software and version via `-sV` |
| Vuln scripts (`--script=vuln`) | Ran against the full top-1000 TCP port scan; results logged to file for follow-up triage rather than acted on blind |
| AitM proxy redirect | Successful — victim's browser traffic routed through attacker-controlled Kali proxy after running the malicious `.bat` |
| Credential interception | Successful — a failed login POST to `/rest/user/login` (HTTP 401) captured the victim's email and password in cleartext in the request body |

The core issue in both halves is the same: unauthenticated banner/version disclosure makes it trivial for an outsider to identify exactly what's running on a host, and unencrypted/unvalidated user behavior (accepting an unsigned script, trusting an internal-looking email, using an unencrypted or improperly validated proxy path) turns a network position into full credential theft. Neither of these outcomes required exploiting a software vulnerability — they exploited configuration exposure and human trust.

## What I'd do differently / lessons learned

- On the scan side, I'd want to correlate the version-banner results against a CVE database (rather than just eyeballing the vuln script output) to prioritize which findings are actually exploitable versus informational noise — that triage step is where a lot of the analyst judgment actually lives.
- On the AitM side, doing this by hand (manually sorting and scanning Burp's HTTP History for one specific request) doesn't scale. In a real environment I'd want to script the extraction of credential-looking POST bodies, or filter more aggressively by content type and status code up front instead of eyeballing a sorted list.
- I'd also want to practice writing this up the way I'd hand it to a SOC lead: not "I ran X and it worked," but a clear statement of business risk (plaintext credentials in transit, unrestricted local script execution, unauthenticated proxy config changes) with a concrete, prioritized remediation list.

## Why this matters for the job

This is essentially a compressed version of a real external assessment: reconnaissance and fingerprinting first, then proving out an actual attack path a real threat actor could use, in this case social engineering plus a AitM position. A SOC/detection-engineering analyst needs both halves — the ability to read a vulnerability scan and prioritize real risk from noise, and the awareness that a lot of real-world breaches don't start with a clever exploit at all, they start with a user clicking something they shouldn't have.
