# Removing Unneeded Device Drivers to Harden a Windows System

**Domain:** Vulnerability Management, System Hardening
**Tools used:** Windows Device Manager (devmgmt.msc)
**Type:** CompTIA hands-on lab (WGU D483 Security Operations, CySA+ CS0-003)

## Scenario

I'm working as a cybersecurity analyst on a Windows Server system. Device
drivers can build up on a machine over time, and some of them don't need to
be there anymore. An old or unwanted driver isn't just clutter. It can hurt
performance, or it can be a way for something that shouldn't be trusted to
sneak onto the system. This lab walks through the tools Windows gives an
admin to manage devices and drivers: scanning for hardware changes,
updating a driver, disabling and re-enabling a device, and removing a
device driver completely.

## Approach

1. Logged into the Windows Server system as an administrator and opened
   Device Manager.
2. Right-clicked a device category and ran Scan for hardware changes.
   Windows checks for any new hardware and installs drivers for it
   automatically if it finds any.
3. Expanded DVD/CD-ROM drives and picked the Microsoft Virtual DVD-ROM
   device to update its driver, using Search automatically for updated
   driver software. Since this system had no internet access, Windows
   could only check its local driver store, not Windows Update. It came
   back saying the best driver was already installed.
4. Disabled the Microsoft Virtual DVD-ROM device, then turned around and
   enabled it again. This is the same first move a lot of troubleshooting
   starts with, turn a device off then back on, to see if that clears up
   whatever was wrong with it.
5. Uninstalled the device completely. Once it was removed, the device
   itself and its whole category (DVD/CD-ROM drives) disappeared from
   Device Manager, since it was the only device in that category.
6. Ran Scan for hardware changes again. Windows found the DVD-ROM
   hardware was still physically present, so it reinstalled the driver on
   its own and the device came back.

## Findings

Windows treats these four actions very differently:

- Scan for hardware changes just looks for anything new and installs
  drivers for it. It doesn't touch devices that are already there.
- Update driver only replaces the driver files. It doesn't change whether
  the device is enabled or connected.
- Disable turns a device off in software, but Windows still knows the
  hardware is there. It survives a reboot in its disabled state, and it
  stays disabled through a rescan too, because Windows isn't looking for
  new hardware, it's looking at hardware it already knows about.
- Uninstall removes the driver and the device entry entirely. But if the
  hardware is still physically connected, Windows finds it again on the
  next scan (or the next reboot) and reinstalls the driver automatically.

The lab's knowledge check gets at the real point here: disabling a device
is the one action that reliably keeps it from being used, even through a
reboot or a rescan, as long as it stays disabled. Uninstalling feels more
permanent, but for a device that's actually still connected, it isn't.
Windows just puts it right back.

## What I'd do differently / lessons learned

The uninstall-then-rescan result is the part that actually matters. I went
in expecting uninstall to be the final answer for getting rid of a device,
and it isn't, not for hardware that's still physically there. If I ever
need to permanently block a device on a Windows system, an unauthorized
USB device, for example, disabling it is the move that actually holds, not
uninstalling it. A better long-term control on a real system would be a
Group Policy setting that blocks the device class outright, so it can't
just get re-enabled by whoever is sitting at the keyboard.

## Why this matters for the job

This maps to a real hardening task. A SOC analyst finds a device or driver
on a system that shouldn't be there, maybe planted through a USB device,
or leftover from an old configuration. Knowing that uninstalling isn't a
permanent fix by itself, and that disabling (or better, a policy-level
block) is what actually keeps a device from coming back, is the difference
between a fix that looks done and one that actually holds.
