# File Carving a Corrupted FAT Image by Rebuilding the Boot Sector

**Domain:** Incident Response, Digital Forensics
**Tools used:** mount, fdisk, fiwalk, fsstat, mmls, testdisk
**Type:** CompTIA hands-on lab (WGU D483 Security Operations, CySA+ CS0-003)

## Scenario

A third drive image, a third suspected case of data destruction. This time the damage goes deeper than a deleted file, the drive image won't even mount normally, which means something at the filesystem structure level itself is broken or was deliberately destroyed. The goal is to recover whatever can still be pulled from an image that every normal filesystem-aware tool refuses to read.

## Approach

1. Tried to mount the image directly and got an error immediately, unlike the last lab where the mount succeeded and just showed an empty looking folder. A failed mount means the filesystem structure itself is unreadable, not just that files were deleted.
2. Ran fdisk against the image. It came back with a disk identifier of all zeros and no partition table at all, confirming something fundamental was missing or destroyed at the very start of the disk.
3. Ran fiwalk for a broader analysis. It returned a long list of errors, most of them "Possible encryption detected." That label is really an entropy check, high randomness in the data, and corrupted or destroyed filesystem structures can produce that same random looking pattern without anything actually being encrypted. I treated that as a flag to verify, not a conclusion to accept.
4. Tried fsstat with a guessed file system type (FAT16), which returned an invalid magic value error, meaning the specific structure fsstat needs to even start reading (the boot sector) was corrupted beyond what fsstat can work around.
5. Ran mmls to try to get partition offsets the way I had in the previous lab. It returned nothing at all this time, confirming there was no usable partition table to read offsets from.
6. Opened the image in testdisk's interactive mode instead of its usual command line reporting mode, since every metadata based tool had failed by this point. Told testdisk to treat the disk as having no existing partition table, then manually specified FAT16 as the filesystem type to try, since that's what the earlier failed fsstat attempt suggested.
7. Tried testdisk's Undelete function first. It came back empty, "No file found, filesystems may be damaged," confirming there wasn't enough intact structure left to just list files normally, even inside testdisk.
8. Used testdisk's boot sector rebuild function instead. It compared what little of the boot sector it could still read against what a standard FAT16 boot sector should look like, and built a replacement. The tool reported "Extrapolated boot sector and current boot sector are different," meaning it built a working reconstruction rather than recovering the exact original bytes.
9. With that rebuilt boot sector held in memory (the original .dd file itself was never modified), testdisk could suddenly list a full, normal looking file structure. Selected all files and copied them out to an output folder. It reported 15 files copied successfully.
10. Reviewed the recovered files directly, opening several with xdg-open, image files, a PDF, a video file, and a zip archive. One of the recovered images, pumpkin.jpg, showed a cat.

## Findings

Every filesystem metadata aware tool (mount, fdisk, fiwalk, fsstat, mmls) failed against this image because the boot sector itself, the very first structure a FAT filesystem needs to interpret anything else on the disk, was damaged. testdisk's boot sector reconstruction rebuilt a working replacement structure well enough to recover all 15 files that were on the original volume, none of them permanently lost, just inaccessible until the boot sector was rebuilt.

## What I'd do differently / lessons learned

Worth being precise about what actually happened here versus what "file carving" usually means. Classic file carving (tools like foremost or scalpel) ignores filesystem structure completely and scans raw bytes for known file signatures, a JPEG's header bytes, a PDF's header bytes, and pulls content out with zero regard for directories or filenames. What testdisk did here is closer to filesystem repair: it rebuilt just enough of the boot sector to make the existing FAT structure readable again, then a completely normal file listing and copy followed. Both get called "carving" loosely, and both matter, but they're solving the problem in genuinely different ways, and I'd want hands on practice with true signature based carving on an image where the filesystem structure is gone entirely, not just its boot sector, since testdisk's boot sector rebuild trick wouldn't help there.

## Why this matters for the job

Damaging or destroying a boot sector is a real, low effort way to make a drive look unreadable to casual inspection, and a tool that flags it as "possible encryption" instead of "corrupted" can send an investigation in the wrong direction if that label gets trusted without verification. Knowing that a boot sector failure and true encryption can look identical to an entropy check, and having a fallback (boot sector reconstruction) that recovers a fully intact file set instead of assuming the data is gone, is the difference between closing a data destruction case with real evidence and closing it with "recovery wasn't possible."
