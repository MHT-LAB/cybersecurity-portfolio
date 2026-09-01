# Recovering Deleted Files from an NTFS Image with tsk_recover

**Domain:** Incident Response, Digital Forensics
**Tools used:** mount, The Sleuth Kit (tsk_recover)
**Type:** CompTIA hands-on lab (WGU D483 Security Operations, CySA+ CS0-003)

## Scenario

A different incident, a different drive image. This time the IRT believes the suspect deliberately destroyed evidence before the drive was seized. I'm handed a raw image of that drive and asked to check whether anything can still be recovered even though the files themselves are gone from the live filesystem.

## Approach

1. Mounted the image directly to a new mount point and listed its contents. The only thing visible was a System Volume Information folder, a normal Windows system folder, nothing else. That's consistent with files being deleted rather than the filesystem itself being damaged, since a normal mount succeeded.
2. Ran tsk_recover against the image, pointed at a new output folder. tsk_recover is a Sleuth Kit tool built specifically to scan an NTFS Master File Table for records still marked as deleted and rebuild any file whose data clusters haven't been overwritten yet, including files that were nested in folders that were also deleted, since the folder records survive in the MFT the same way file records do. It reported 8 files recovered.
3. Listed the recovered output folder and found some files at the top level and a folder structure underneath it, dir1 and dir1/dir2, both of which tsk_recover rebuilt from the MFT's own folder records even though those folders no longer existed on the live filesystem.
4. Checked inside those recovered subfolders specifically, since the lab's own follow-up question was about which recovered files were nested rather than sitting in the output root. frag3.dat and mult2.dat were the two files found inside the subfolder structure.

## Findings

tsk_recover pulled back all 8 files NTFS still had a deleted-but-not-yet-overwritten record for, including full folder structure for files that had been organized into now-deleted subfolders. The recovered files in this particular test image don't carry real content, this is a testing image built to prove the recovery mechanism works, not a real case, but the recovery itself was complete and automatic.

## What I'd do differently / lessons learned

The real work in a case like this starts after recovery, not at it. tsk_recover did the hard part automatically here because it's built for exactly this scenario, but a real investigation would need timestamp analysis, hash comparison against known file sets, and content review on every recovered file to figure out what actually mattered. I'd want practice tying a tsk_recover pass into a broader triage script that hashes everything it pulls back and flags files worth a closer look first.

## Why this matters for the job

NTFS keeps enough of its own bookkeeping around after a delete that automated recovery like this works far more often than most people expect, deletion is not destruction on NTFS unless the data clusters actually get overwritten. That's directly useful in an insider threat or data destruction case: a suspect who just hits delete and empties the recycle bin, thinking that's enough, often leaves a fully recoverable MFT record behind. Running a tool like tsk_recover as an early, low cost step before assuming data is actually gone is standard practice, and it's often the difference between a case with evidence and one without.
