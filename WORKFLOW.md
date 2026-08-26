# Workflow — turning a lab into a portfolio writeup

1. **Finish the lab** in WGU's D483 CompTIA lab environment as normal (for
   CySA+ study/gate credit).
2. **Bring it to Claude same-day** (or next session): paste what you did —
   the scenario in your own words, tools you used, what you found, any
   screenshots of your own output/terminal/results (not the lab platform's
   UI/instructions).
3. Claude discusses the material with you along the way (what it means,
   why it's done that way, how it maps to a real SOC job) — this doubles as
   a study/learning pass, not just writeup production.
4. Claude drafts a writeup using `TEMPLATE-lab-writeup.md`, in your voice,
   with a "why this matters in the real world" section, no copied vendor
   scenario/question text (your own commands and tool syntax are fine to
   keep verbatim), and saves it to `labs/YYYY-MM-DD-slug.md`.
5. Claude updates the index in `README.md`.
6. You review, tweak anything that doesn't sound like you, and push to
   GitHub (`git add . && git commit -m "Add [lab name] writeup" && git push`).

## Notes for future sessions

- This repo lives at `Documents/Claude/Projects/Personal Workflow/cybersecurity-portfolio/`
  on Dustin's Mac — not yet an initialized git repo or pushed to GitHub as of
  2026-08-26. Check `git status` / `git remote -v` before assuming it's live.
- Scope: WGU D483's ~30 CompTIA CySA+ hands-on labs (see
  `Personal/WGU/D483 Security Operations/D483 - Syllabus Reference.md` in the
  Obsidian vault for the full lab list by week). Not the broader
  `Personal/Lab/` practice files (LimaCharlie, BloodHound, etc.) unless
  Dustin says he wants those folded in too.
- Priority framing: cross-reference against the Huntress public-intel scan
  (see project memory `feedback_huntress_intel_scan_resume_trigger.md`) so
  writeups emphasize the skills/tools that show up in real job postings —
  threat hunting, log correlation, IoC analysis, incident response — not
  just whichever lab comes next in the syllabus order.
- Don't reproduce CompTIA/CertMaster lab instructions, question text, or
  platform screenshots — copyright + it's weaker portfolio material anyway.
  Own-words methodology is the whole value proposition.
