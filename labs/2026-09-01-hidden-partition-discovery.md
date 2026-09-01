# Finding a Hidden Partition on a Damaged MBR Disk Image

**Domain:** Incident Response, Digital Forensics
**Tools used:** fdisk, testdisk, fiwalk, The Sleuth Kit (mmls, fsstat, fls, istat), losetup
**Type:** CompTIA hands-on lab (WGU D483 Security Operations, CySA+ CS0-003)

## Scenario

An incident response team investigation stalled because they couldn't find enough evidence. I'm brought in as the company's cybersecurity analyst to do my own forensic pass on a raw drive image pulled from the system involved, since the original machine has already been wiped and put back into service. The goal is to check whether the standard tools missed something on this disk, specifically whether a partition table can hide a logical drive from a tool that only trusts what the table itself reports.

## Approach

1. Ran `fdisk -l` against the image first, the standard starting point for reading a disk's partition layout. It listed six lines: three primary partitions, one Extended partition (a container, not a real formattable volume), and two logical drives inside that extended container. fdisk also threw a warning, "Ignoring extra data in partition table 5," which flagged that the extended boot record for one of the logical drives had leftover data fdisk didn't know how to interpret. That warning turned out to be the first real clue.
2. Did the math on the extended partition's own size against its two listed logical drives. The extended partition was 155232 sectors. The two logical drives inside it added up to only 102690 sectors combined, leaving over 52000 sectors unaccounted for, almost exactly the size of one of the existing logical drives. That gap meant either wasted space, bad sectors, or a partition that exists on disk but isn't listed in the table.
3. Ran testdisk against the same image to get a second opinion, since testdisk scans for filesystem signatures directly instead of only trusting the partition table's own bookkeeping. It found seven entries where fdisk only found six, a logical drive fdisk never reported at all.
4. Ran fiwalk for a broader look at the image. It confirmed the same partitions and showed each one held a single text file named after that partition, useful for later confirming which partition I was actually looking at.
5. Tried fsstat directly against the image without specifying an offset, which failed since fsstat needs a filesystem type and a specific offset to know where to start reading, it can't work against a whole multi-partition disk image blind.
6. Ran mmls (a Sleuth Kit tool) to get the sector offset of every partition and logical drive, including the one testdisk found that fdisk didn't. That gave me the starting sector I needed to point fsstat, fls, and istat directly at the hidden partition.
7. Ran fsstat with the FAT16 file system type and the hidden partition's offset. It returned filesystem details but nothing unusual on its own.
8. Ran fls against the same offset to list what was actually inside that hidden partition. It showed one real file, second-3.txt, along with several system entries.
9. Ran istat against each inode in that partition one at a time. Inode 2 was the root directory. Inode 3 was second-3.txt, a clean entry with no real timestamps, meaning it was placed there deliberately just to prove the hidden partition's existence. Inode 4 was a second file, SECOND-3.txt, that did have real timestamps but no recoverable data, meaning it was left over from something that existed on this volume before it got hidden. Inode 5 came back invalid, meaning I'd reached the end of what the partition actually held.
10. Used losetup to create a loop device from the image file, then mounted the hidden partition (the 7th entry in the partition table, accounting for the extended partition itself taking up a slot) to a new mount point and listed its contents directly. second-3.txt was there. SECOND-3.txt was not, confirming it really had been deleted and its data really was gone.

## Findings

The disk's partition table only reported six formattable volumes: three primary partitions and two logical drives inside the extended partition. A seventh logical drive existed on the physical disk but had been left out of, or stripped from, the partition table, so a tool that only reads the table (like fdisk) never saw it. Recovering the sector offset with mmls and mounting it directly confirmed the hidden partition held one real file and one deleted, unrecoverable file with legitimate timestamps, evidence the volume had prior activity before whatever hid it.

## What I'd do differently / lessons learned

I'd run testdisk or mmls automatically as a standard second pass any time a partition table looks even slightly inconsistent, instead of treating fdisk's output as final. The size-math check (extended partition size vs. the sum of its logical drives) is fast and something I can do by hand in under a minute, and it's what actually pointed me at this before I ran a second tool at all. I'd want to build that gap check into a script for any real case, since doing sector arithmetic by hand doesn't scale past a handful of partitions.

## Why this matters for the job

Hiding a logical drive by stripping its partition table entry is a real technique, not just a lab trick, and any tool that only trusts the partition table's own claims will walk right past it. This is the exact reason a real forensic process never stops at one tool's output. fdisk is fast and normal for routine checks, but a tool like testdisk or mmls that reads raw filesystem signatures instead of just the table's bookkeeping is what actually catches a partition someone tried to hide.
