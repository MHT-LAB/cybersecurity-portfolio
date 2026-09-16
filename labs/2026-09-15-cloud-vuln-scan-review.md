# Analyzing Cloud Vulnerabilities: Auditing AWS Environments with ScoutSuite and Prowler

**Domain:** Threat and Vulnerability Management
**Tools used:** ScoutSuite, Prowler, AWS IAM, AWS CloudTrail, AWS Config, AWS VPC
**Type:** Hands-on lab (Security Operations)

## Scenario

This lab had two parts built around the same core skill: reading an automated cloud security scan and mapping the results back to a written security policy. In the first part, I played a security analyst at a company that had recently moved infrastructure to AWS, tasked with running a ScoutSuite scan and checking the results against a supervisor-provided policy covering account use, monitoring, and network access. In the second part, the scenario shifted to an M&A context — my company was acquiring a call-center business, and I was on the integration team reviewing a Prowler scan the acquired company had provided as part of due diligence, looking for risks that would need to be remediated before the two environments were merged.

## Approach

**Part 1 — ScoutSuite audit against the internal policy**

I started at the ScoutSuite dashboard rather than diving into individual services, since the dashboard's per-service Danger/Warning/Good counts tell you where to spend your time before you start clicking into details. The dashboard showed CloudTrail, EC2, and IAM all flagged Danger, with Config and VPC flagged Warning — S3, RDS, and the load balancer services came back clean. EC2 (116 findings) and VPC (200 findings out of 257 checks) had the highest raw counts, which pointed me toward the network side of the policy checklist as a priority area even before I'd looked at specifics.

From there I worked through the policy checklist section by section rather than the ScoutSuite service list, since the checklist was what I actually had to report against:

- *Account use* — I drilled into the IAM findings dashboard and matched dangers against the policy line items one at a time (root account use, MFA on privileged accounts, root access keys, password policy).
- *Monitoring* — I checked the VPC and Config dashboards for the flow-logging and config-recorder requirements, and the CloudTrail dashboard for the activity-monitoring requirement.
- *Network* — I checked the EC2 security group findings for the RDP-from-JumpBoxes-only requirement, since RDP exposure is where an "essential traffic only" policy break shows up first.

**Part 2 — Prowler review for the acquisition**

The acquired call center handed over a Prowler report rather than raw console access, which is a common situation in an M&A context — you often only get what the other side is willing to share, and a report is faster for them to produce than access. Prowler's output is CLI-style pass/fail text organized by CIS AWS Foundations Benchmark control number rather than a clickable dashboard, so my approach here was different: I searched the report by keyword (MFA, Flow Log, security group, encryption) to jump straight to sections relevant to the risks the integration team would care about, rather than reading it top to bottom. Given the acquisition context, I focused on things that would represent immediate exposure once the two environments were connected — anything internet-facing, anything touching credentials, and anything that would break the acquiring company's own compliance posture the moment the accounts were linked.

## Findings

**Part 1 — ScoutSuite (internal environment)**

| Policy Area | Requirement | ScoutSuite Finding | Status |
|---|---|---|---|
| Account Use | No root/admin use for routine tasks | Root Account Used Recently, Root Account Logon Detected | Non-compliant |
| Account Use | Password length/complexity/expiration/reuse | Minimum Password Length Too Short, Password Expiration Disabled, Password Policy Allows Reuse of Passwords, Passwords Expire after 90 Days (failing) | Non-compliant (4 separate password-related dangers) |
| Account Use | MFA on privileged accounts | Privileged Accounts configured without MFA, Root Account without Hardware MFA | Non-compliant |
| Account Use | No root access keys | Root Account Has Active Keys | Non-compliant |
| Account Use | (additional) | Managed Policy Allows All Actions — an overly permissive managed policy, worth flagging even though it wasn't a named line item in the policy | Non-compliant |
| Monitoring | VPC Flow Logging | Subnet without a Flow Log | Non-compliant |
| Monitoring | Config Recorder tracking changes | AWS Config Not Enabled | Non-compliant |
| Monitoring | CloudTrail monitoring account activity | CloudTrail showed 16 findings across 16 applicable checks (100% flagged) | Non-compliant — the dashboard confirmed a problem across every CloudTrail check, though I didn't capture the individual check name for the writeup |
| Network | RDP restricted to JumpBoxes | Security Group Opens RDP Port to All | Non-compliant |

**Part 2 — Prowler (acquired call center environment)**

- **MFA gap at scale:** the majority of "student-XX" style accounts had console passwords enabled with MFA disabled — this wasn't one or two stray accounts, it was the norm across the credential report.
- **Root account:** no access keys existed for root (a pass), and MFA was enabled — but only virtual MFA, not hardware MFA, which fails the stricter check.
- **IAM policy hygiene:** managed and inline policies were attached directly to individual users instead of through groups or roles, making access harder to audit and revoke cleanly.
- **No AWS Config recorder enabled** — same gap as the internal environment, meaning configuration drift in the acquired account wouldn't be tracked either.
- **VPC Flow Logs not found** in the regions checked.
- **CloudTrail alerting gap:** no CloudWatch log metric filters or alarms existed for a long list of CIS-recommended events — console auth failures, S3 bucket policy changes, Config configuration changes, security group changes, NACL changes, network gateway changes, route table changes, and VPC changes. CloudTrail may have been logging, but nothing was watching the logs in real time.
- **Security groups open to the internet:** several specifically-named security groups in us-east-1 allowed both SSH (22) and RDP (3389) inbound from 0.0.0.0/0. This is the most urgent finding of the whole assessment given the acquisition context — remote administrative access wide open on the public internet is the kind of thing that needs fixing before, not after, network integration.
- **Unencrypted EBS volumes** found in us-east-1.
- **Expired ACM certificate** (example.com), well past its expiration date, still present in the account.
- **S3 bucket logging disabled** across every bucket in the account — none had server access logging turned on, which would make any future incident investigation on those buckets much harder.
- **No incident support role configured**, meaning there wasn't a designated AWS Support-facing role for incident response.

## What I'd do differently / lessons learned

Working from the ScoutSuite dashboard down to specific checks was faster than clicking through every service, and I'd start there every time rather than getting pulled into the first Danger flag I see — the raw finding counts on the dashboard are a decent proxy for where the real risk concentration is. For Prowler, I underestimated how much slower plain-text CLI output is to review compared to a clickable dashboard; keyword search helped, but for a real assessment I'd pipe Prowler's JSON/CSV output into a spreadsheet or a simple script to sort by severity and CIS control number rather than scrolling a PDF. I'd also automate the cross-reference step — matching scanner findings to policy line items by hand works fine for eight policy items, but it doesn't scale, and a real SOC would want that mapping built into a dashboard or ticketing rule rather than done manually every time a scan runs.

One thing I'd tighten up in my own process: I lost track of which specific CloudTrail check was failing in the ScoutSuite report because I didn't screenshot it before moving on. For a real audit deliverable, I'd want to capture every finding I'm going to cite before I move to the next section, since going back to re-verify wastes time and risks citing something from memory instead of the actual report.

## Why this matters for the job

This is the bread-and-butter workflow of a lot of GRC and detection-engineering-adjacent SOC work: an automated scanner produces a wall of findings, and the analyst's real job is translating that wall into "does this violate our policy, and how bad is it" for people who don't want to read raw tool output. The M&A angle in part two is also realistic — due-diligence security reviews of an acquisition target's cloud environment are a common ask, and knowing how to quickly triage someone else's environment from a report alone (without live console access) is a skill that shows up in real security operations and risk management roles.
