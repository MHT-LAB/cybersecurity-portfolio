# Web Vulnerability Scanning Across CLI, TLS, Online, and Automated Recon Tools: Comparing Nikto, Wapiti, Nmap NSE, SSL Labs, Pentest Tools, and Legion

**Domain:** Threat and Vulnerability Management
**Tools used:** Nikto, Wapiti, Nmap (NSE scripting), SSL Labs, Pentest Tools (Website Scanner), Legion, Kali Linux, Firefox
**Type:** Hands-on lab (WGU D483 Security Operations)

## Scenario
I ran a set of web-focused vulnerability scanners against multiple targets to compare how different tools — CLI-based, TLS-focused, online SaaS scanners, and an automated multi-tool recon framework — approach the same general problem from different angles. On a DVWA instance, I compared a broad server-focused scanner (Nikto) against an application-focused scanner (Wapiti). On a second target, I used Nmap's scripting engine (NSE) to check TLS/SSL configuration against a company policy. Since the lab environment's own targets aren't internet-exposed, I then switched to my local browser and ran two SaaS-based scanners — Qualys SSL Labs and Pentest Tools' Website Scanner — against a public test target (badssl.com) built specifically to have known-bad TLS configurations. Finally, I used Legion, a GUI-driven recon framework that chains together nmap, Nikto, Vulners, and roughly 100 other scripts automatically, to do broad service/version enumeration and CVE lookups against a lab target without having to run each underlying tool by hand. The point across all of it wasn't just running tools; it was learning which tool answers which kind of question, how their grading/severity models differ, and how to triage findings that come from very different scan types and sources.

## Approach
1. **Recon on the server itself with Nikto.** I ran Nikto against the target and piped the output to a file so I'd have a record to review afterward instead of relying on scrollback. Nikto's strength is server fingerprinting and known-file/known-CGI checks, so I expected it to tell me what web server software and version was running, plus flag any stale or risky default content.
2. **Reviewed the Nikto output.** The scan identified the server as **Apache 2.4.41** — that's the first thing I'd want to confirm manually (e.g., banner grab with `curl -I` or `netcat`) since server banners can be spoofed or left over from a prior config. Beyond the version fingerprint, Nikto flagged a handful of missing security headers and information-disclosure issues (detailed below).
3. **Switched to Wapiti for application-layer testing.** Wapiti needs a full URL (protocol included), so I pointed it at `http://` the target rather than just the hostname. Where Nikto looks at the server and known paths, Wapiti actually crawls the app and tests input handling and response headers for injection-class issues and security header hygiene.
4. **Reviewed the generated HTML report in Firefox** rather than just the terminal output, since Wapiti's report format groups findings by category and severity, which made it easier to compare against what Nikto had already found.
5. **Cross-referenced the two result sets.** Both tools flagged missing/weak security headers, but from different angles (Nikto: what's absent on the response; Wapiti: what's absent *and* what that absence enables). That overlap is useful — when two independent tools agree on a header gap, it's a stronger signal than either one alone.
6. **Checked TLS configuration on a second target with Nmap NSE.** Nikto and Wapiti are good at HTTP-layer issues, but neither is built to enumerate supported TLS/SSL versions and cipher suites — that's a job for Nmap's `ssl-enum-ciphers` script. I ran it with maximum verbosity against the second target and saved the output to a file rather than just reading it off the terminal, since this is exactly the kind of result I'd want to attach to a ticket.
7. **Compared the result against company policy.** The stated policy was TLSv1.2/TLSv1.3 only. Rather than eyeballing the raw cipher list, I checked specifically for the presence of any TLSv1.0/TLSv1.1 (or SSLv3) entries in the script output, since that's the direct pass/fail criterion the policy sets.
8. **Moved to online scanners since the lab targets weren't internet-exposed.** Neither of the two lab-hosted sites could be reached by an internet-based scanning service, so for the SaaS-based tools I switched targets to `badssl.com` — a site purpose-built with a range of known-bad TLS/certificate configurations for testing exactly this kind of tool.
9. **Ran an SSL Labs evaluation.** SSL Labs grades a server across four categories — Certificate, Protocol Support, Key Exchange, and Cipher Strength — and rolls them up into a single letter grade. I read through badssl.com's breakdown, then, to build intuition for what "good" vs. "bad" looks like in the tool, pulled up a recently-scanned A/A+ site from the "Recent Best" list and a recently-scanned F/T site from "Recent Worst" list and compared what specifically drove each grade.
10. **Ran a Pentest Tools Website Scanner (Light) scan** against the same target. This tool takes a different approach from SSL Labs — instead of a single letter grade, it buckets every discovered issue into a severity label (Critical / High / Medium / Low / Info), which is closer to how a vulnerability management ticketing system would categorize findings.
11. **Ran Legion against a lab target for broad service/version enumeration.** Legion is a GUI wrapper that chains nmap, Nikto, Vulners, and a large set of other scripts (close to 100) into one automated pass, rather than me manually invoking each tool. After adding the host, I let the initial nmap scan run (its time estimate was well short of reality) and then worked through the interface's tabs — Services, Scripts, Information, and CVEs — as they populated, rather than waiting for a single "done" state, since Legion fills these in progressively and some tabs (Scripts, CVEs) lag well behind the initial service enumeration.
12. **Recorded the discovered service fingerprint** rather than the full CVE list, since the point of this step was evaluating what an automated recon pass surfaces about a target's exposed attack surface before any deeper manual investigation.

## Findings

**Server fingerprint (Nikto):** Apache 2.4.41

**Nikto findings:**

| Finding | Why it matters |
|---|---|
| Anti-clickjacking `X-Frame-Options` header not present | App can potentially be framed by a malicious site (clickjacking) |
| `PHPSESSID` cookie created without `HttpOnly` flag | Session cookie is readable by client-side JS — raises session-theft risk if XSS exists anywhere on the app |
| OSVDB-3268 — Directory indexing found | A directory is browsable, which can expose file structure or files not meant to be listed |
| Configuration information may be available remotely | Server may be leaking config details that aid an attacker's recon |
| OSVDB-630 — Web server may reveal its real IP address in headers | Could undermine any reverse-proxy/CDN fronting meant to hide the origin server |

**Wapiti findings:**

| Finding | Why it matters |
|---|---|
| CSP (Content-Security-Policy) is not set | No browser-enforced restriction on script/resource sources — raises impact of any XSS |
| `X-Frame-Options` is not set | Confirms the clickjacking exposure Nikto also flagged |
| Strict-Transport-Security is not set | If the app is ever reached over HTTPS, browsers won't be forced to stay on HTTPS for future visits |
| Secure flag is not set on the cookie | Session cookie could be transmitted over plain HTTP, exposing it to interception |

Notably, neither tool reported an actual working XSS or SQLi proof-of-concept against this target — both flagged *header and cookie-attribute weaknesses* rather than a demonstrated injection vulnerability. That distinction matters for triage: these are hardening gaps, not confirmed active exploits.

**TLS configuration (Nmap NSE, second target):** The scan showed the site does **not** comply with the TLSv1.2/TLSv1.3-only policy — both **TLSv1.0 and TLSv1.1 were still enabled** alongside the modern versions. That's a real finding, not a hardening suggestion: TLSv1.0/1.1 are deprecated protocol versions with known weaknesses (e.g., susceptibility to downgrade and padding-oracle-style attacks depending on cipher support), and most compliance frameworks (PCI-DSS included) require they be disabled. This is the kind of finding I'd flag as higher priority than the header gaps above, since it's a direct, unambiguous policy violation rather than a "would be nice to add" hardening item.

**SSL Labs (badssl.com):** SSL Labs' grade is built from four test categories — **Certificate, Protocol Support, Key Exchange, and Cipher Strength.** Since badssl.com is purpose-built to demonstrate broken TLS configurations, the site's own overall rating isn't meaningful on its own (different subdomains on badssl.com are intentionally misconfigured in different ways to serve as test cases). The more useful exercise was comparing a real A/A+-rated site against a real F/T-rated site pulled from SSL Labs' own "Recent Best"/"Recent Worst" lists: the poorly-rated site's report highlighted its worst issue in red/orange, which is a fast visual way to see what's actually driving a bad grade (in the case I reviewed, this pointed to weak protocol/cipher support rather than a certificate problem) — a useful reminder that "TLS is broken" can mean several very different underlying issues.

**Pentest Tools Website Scanner (Light, badssl.com):** This scanner organizes every finding into one of five severity buckets — **Critical, High, Medium, Low, Info.** That's a more actionable structure for triage than a single letter grade, since it maps more directly onto how a ticket would be prioritized in a real vuln management workflow (fix Critical/High first, track Low/Info for a later hardening pass).

**Legion (515support.com target):** The automated scan enumerated a mixed set of exposed services and versions:

| Service | Version |
|---|---|
| OpenSSH | 8.4p1 |
| Postfix smtpd | — |
| Apache httpd | 2.4.54 |
| Docker Registry | API 2.0 |
| Dovecot imapd | — |

The combination is notable on its own even before cross-referencing CVEs: this single host is exposing SSH, mail (Postfix + Dovecot), a web server, and a Docker Registry API all at once. From a triage standpoint, an exposed Docker Registry API is the item I'd want to check first — an unauthenticated or weakly-authenticated registry endpoint can leak image contents or, worse, allow image pushes, which is a much bigger blast-radius issue than an outdated mail daemon version. I'd treat Legion's output here as a lead list for manual verification (confirm auth requirements on the registry endpoint, check each version against current CVE feeds) rather than as a finished report — its value is breadth and speed, not final-answer accuracy.

## What I'd do differently / lessons learned
- I'd script a quick header-comparison step (e.g., `curl -I` before and after remediation) so I'm not relying purely on scanner output to confirm a fix landed.
- Running Nikto and Wapiti back-to-back against the same target manually works fine for a single host, but in a real SOC this is exactly the kind of recurring scan that should be scheduled and diffed automatically — I'd rather see a ticket auto-generated when a *new* finding appears than re-read a full report every time.
- I'd want to validate the directory indexing and config-disclosure findings by hand (not just trust the scanner) before writing them into a report, since Nikto in particular is prone to some false positives on older/rebranded checks (OSVDB references are from a discontinued database, which is a good reminder to double check current CVE/CWE mappings rather than just carrying forward the label the tool gives you).
- I'd get more practice reading Wapiti's HTML report structure quickly — the terminal output is compact, but the report groups by vulnerability class, which is more useful for a written finding but takes a bit longer to navigate cold.
- For the TLS check, I'd want to build a habit of running `ssl-enum-ciphers` (or an equivalent like `testssl.sh`) as a standard step any time a web target is in scope, rather than treating it as a separate one-off scan — protocol-version compliance is a fast, high-value check that's easy to skip if it's not part of the routine.
- Comparing a top-rated and bottom-rated site side by side in SSL Labs was more useful than just reading badssl.com's own report in isolation — I'd build that "known good vs. known bad" comparison habit into how I review any new tool's output, since it calibrates what a report should look like before you're staring at an ambiguous real-world result.
- I'd want to get comfortable mapping severity labels across tools (Nikto/Wapiti don't use Critical/High/Medium/Low the way Pentest Tools does), since a real environment will have multiple scanners feeding into one prioritized backlog, and someone has to normalize those severities before they're comparable.
- Legion's tabs populate at very different speeds (Services fills in fast, Scripts and CVEs lag well behind), so I'd need to build patience into how I use it — checking back periodically rather than assuming an empty tab means "nothing found." I'd also want to spot-check a couple of its CVE matches manually before trusting them wholesale, since automated CVE-to-version matching can misfire on point releases or backported patches.

## Why this matters for the job
This is the kind of first-pass triage a SOC analyst or vuln management team does constantly: run automated scanners across multiple layers (application, server, transport) and multiple tool types (CLI, scripting-engine, SaaS, automated recon frameworks), separate confirmed issues from noise, and hand prioritized, verified findings — not raw tool output — to whoever owns remediation. Knowing that a header gap, a deprecated-TLS-version finding, and an exposed Docker Registry API don't all carry the same urgency is exactly the kind of judgment call that separates a useful report from a raw scan dump.
