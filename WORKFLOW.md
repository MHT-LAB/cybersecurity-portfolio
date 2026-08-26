# Workflow: turning a lab into a portfolio writeup

1. **Finish the lab** in WGU's D483 CompTIA lab environment as normal (for
   CySA+ study and gate credit).
2. **Bring it to Claude same-day** (or next session). Paste what you did:
   the scenario in your own words, tools you used, what you found, and any
   screenshots of your own output, terminal, or results (not the lab
   platform's UI or instructions).
3. Claude discusses the material with you along the way (what it means,
   why it's done that way, how it maps to a real SOC job). This doubles as
   a study and learning pass, not just writeup production.
4. Claude drafts a writeup using `TEMPLATE-lab-writeup.md`, in your voice,
   with a "why this matters in the real world" section, and no copied
   vendor scenario or question text (your own commands and tool syntax are
   fine to keep verbatim). Saves it to `labs/YYYY-MM-DD-slug.md`.
5. Claude updates the index in `README.md`.
6. Claude commits and pushes to GitHub directly. Push access is already
   set up, so you don't need to run any git commands yourself.

## Notes for future sessions

- This repo lives at
  `Documents/Claude/Projects/Personal Workflow/cybersecurity-portfolio/` on
  Dustin's Mac, with a git remote pointed at
  `github.com/MHT-LAB/cybersecurity-portfolio` (public). Push access is
  configured via GitHub CLI in the Cowork device shell. See project memory
  `project_lab_writeups_job_hunting.md` for the auth setup details if a
  future session can't push.
- Scope: WGU D483's roughly 30 CompTIA CySA+ hands-on labs (see
  `Personal/WGU/D483 Security Operations/D483 - Syllabus Reference.md` in
  the Obsidian vault for the full lab list by week). Not the broader
  `Personal/Lab/` practice files (LimaCharlie, BloodHound, etc.) unless
  Dustin says he wants those folded in too.
- Priority framing: cross-reference against the Huntress public-intel scan
  (see project memory `feedback_huntress_intel_scan_resume_trigger.md`) so
  writeups emphasize the skills and tools that show up in real job
  postings, like threat hunting, log correlation, IoC analysis, and
  incident response, not just whichever lab comes next in the syllabus
  order.
- Don't reproduce CompTIA/CertMaster lab instructions, question text, or
  platform screenshots. It's a copyright issue, and it's weaker portfolio
  material anyway. Own-words methodology is the whole value proposition.
- **Voice rule:** no em dashes and no semicolons in anything meant to read
  as Dustin's voice (the writeup content and README framing especially).
  Use periods and commas instead. Same rule his WGU coursework follows,
  see project memory `feedback_wgu_voice.md`.
