# Removing Unneeded Applications and Services

**Domain:** Vulnerability Management, System Hardening (Attack Surface Reduction)
**Tools used:** Windows Programs and Features, Server Manager (Remove Roles and Features Wizard)
**Type:** CompTIA hands-on lab (WGU D483 Security Operations, CySA+ CS0-003)

## Scenario

I'm working as a cybersecurity analyst on a Windows Server system, cleaning
up things that don't serve a real business need anymore. Two targets: an
old diagnostic app called CPUID CPU-Z that nobody uses, and the FTP
service running as part of IIS, an insecure legacy protocol that has no
reason to be exposed if the business isn't actually using it.

## Approach

1. Opened Programs and Features, selected CPUID CPU-Z 2.01 from the
   list, and chose Uninstall. Confirmed the removal and watched the
   progress window until it finished.
2. Switched to Server Manager to deal with FTP. This is a role service,
   not a regular app, so it doesn't show up in Programs and Features the
   same way.
3. Went to Manage, then Remove Roles and Features. This wizard is the
   only one that can actually remove a role or feature on a server. The
   "Turn Windows features on or off" option elsewhere just opens the Add
   Roles and Features Wizard, which only adds things, it can't remove
   them.
4. Expanded Web Server (IIS) in the roles list and cleared the checkbox
   for FTP Server, leaving everything else selected as-is.
5. On the confirmation page, the wizard listed exactly what was about to
   be removed: Web Server (IIS), FTP Server, FTP Service. That's the
   checkpoint to catch a mistake before committing to it, like accidentally
   pulling the whole IIS role instead of just the FTP piece.
6. Checked the box to restart the server automatically, since removing a
   role service like this needs a reboot to fully take effect, confirmed
   the warning, and selected Remove. The server rebooted on its own once
   the removal finished.

## Findings

- Adding and removing roles/features on Windows Server go through two
  different wizards. Add Roles and Features Wizard installs things.
  Remove Roles and Features Wizard takes them away. Programs and
  Features' "Turn Windows features on or off" link only routes to the
  add side, even though it looks like a general toggle.
- The confirmation page before a removal matters. It spells out precisely
  what's being touched, which is the difference between removing just the
  FTP Server role service and accidentally taking down the whole Web
  Server (IIS) role that other things might depend on.
- Removing a server role can force a reboot, unlike uninstalling a normal
  desktop app, which is why the wizard has its own restart option built
  in.

## What I'd do differently / lessons learned

The CPU-Z removal was low risk, a normal app uninstall. The FTP removal
needed more care, since it lives under a shared IIS role tree where
picking the wrong checkbox could take out something the server actually
needs. Reading the confirmation page before hitting Remove mattered here
the same way reading a script before running it mattered in an earlier
lab. If I ever need to bring FTP back for a legitimate reason, I know now
it's the Add Roles and Features Wizard I'd reach for, not the Remove one,
since those two tools stay separate on server systems.

## Why this matters for the job

This is basic attack surface reduction. Any app or service that isn't
tied to an actual business need is one more thing that could carry a
vulnerability and one more thing that needs patching. FTP is a specific,
well-known finding in a hardening review, since it sends credentials in
cleartext, and the right fix isn't to just disable it, it's to remove the
role entirely so it can't get switched back on by accident later.
