# Recovering Hidden JPEG Evidence with Autopsy

**Domain:** Incident Response, Digital Forensics
**Tools used:** md5sum, Autopsy (The Sleuth Kit GUI)
**Type:** CompTIA hands-on lab (WGU D483 Security Operations, CySA+ CS0-003)

## Scenario

A suspect is believed to have hidden evidence photos on a drive using a common naming pattern (fileN) while trying different tricks to keep each one from being found easily, wrong extensions, deletion, and burying one inside an archive. I'm handed a raw image of the drive and asked to find and recover as many of the hidden JPEGs as I can, using Autopsy end to end, from acquiring the image through exporting the actual evidence files.

## Approach

**Acquisition and integrity verification**

1. Copied the test image and its published hash off the lab's ISO media, extracted it, and confirmed the file size matched what was expected before doing anything else.
2. Hashed the extracted image with md5sum and compared it against the hash published alongside the original test file. A match confirms the copy I'm about to analyze is bit for bit identical to the source, before any analysis starts.

**Case setup**

3. Launched Autopsy and created a new case, recording an investigator name, host name, host description, and time zone. This is the chain of custody documentation, not busywork, it's the record of who analyzed what evidence and when.
4. Imported the image using the Symlink option rather than Copy or Move, so Autopsy works from a reference to the original file instead of ever touching it directly. Autopsy recalculated the MD5 hash on import, and it matched the hash I'd already confirmed manually, a second independent integrity check baked into the tool itself.

**Finding file1.jpg (known filename, direct search)**

5. Searched the file listing directly for file1.jpg by name and found it at C:/alloc/file1.jpg. Before trusting it, checked the ASCII Strings and Hex views. The JFIF string and the FFD8 FFE0 header bytes at the start of the file confirmed it was a real JPEG, not just a plausible name. Exported it, renamed it back to file1.jpg, and opened it to confirm.

**Finding file2 (wrong extension)**

6. Browsed the alloc/ directory directly instead of searching by name and found file2.dat sitting next to file1.jpg. Autopsy's preview pane rendered it as an image even though the extension said .dat, which was the first clue. The same ASCII Strings and Hex check (JFIF, FFD8 FFE0) confirmed it really was a JPEG with a deliberately wrong extension, meant to defeat a simple filename or extension based search. Exported, renamed to file2.jpg, confirmed.

**Ruling out file3, file4, and file5**

7. Checked file3.jpg, which had a plausible name but turned out to be plain text, a message stating it wasn't actually a JPEG, with no JPEG signature at all.
8. Checked file4.jpg, which had a partial match, FFD8 present but not the full FFD8 FFE0 pair, genuinely ambiguous rather than a clean JPEG or a clean non match.
9. Checked file5.rtf, which showed nothing readable and no JPEG signature either.
10. None of these three panned out. Verifying a negative this thoroughly, instead of stopping once enough real hits are found, matters as much as finding the real evidence.

**Finding file6 and file7 (deleted files)**

11. Switched to Autopsy's All Deleted Files view and found file6.jpg under a deleted directory. Signature check confirmed a real JPEG. Exported, renamed, confirmed.
12. In the same deleted files view, found file7.hmm, a deleted file with a completely nonstandard extension. The same signature check (JFIF, FFD8 FFE0) confirmed it was also a real JPEG, combining two evasion techniques, deletion and extension tampering, on the same file. Exported, renamed to file7.jpg, confirmed.

**Finding file8 (inside an archive)**

13. Expanded the directory tree and found file8.zip in an archive/ folder. Recognized the ZIP format's own signature, PK at the start (hex 504B), before even extracting anything. The ASCII Strings view showed file8.jpg's name inside the archive's own internal listing.
14. Exported the zip, extracted it locally, and opened file8.jpg from inside it to confirm.

**Cross-checking with a raw keyword search**

15. Ran a keyword search for the string "file" across the whole image, not limited to the file listing. It returned a hit in a raw disk cluster showing the same PK signature and file8.jpg name found in the archive, confirming the same data independent of the filesystem's own directory structure. A keyword search like this can find content even in a case where the filesystem's own metadata is damaged or incomplete, since it isn't relying on that metadata to know where to look.

## Findings

Recovered 5 of the 10 JPEGs the test image was built to hide (file1, file2, file6, file7, file8), covering four distinct evasion techniques: a normal file, a renamed extension, a deleted file, a deleted file with a renamed extension, and a file buried inside an archive. Every one of them was confirmed the same way, by checking the actual file signature (JFIF string, FFD8 FFE0 header bytes) rather than trusting the filename or extension. Three other files with plausible names (file3, file4, file5) were checked and ruled out using the same method, none of them were real JPEGs.

## What I'd do differently / lessons learned

The lab left five more hidden pictures as an optional self-guided challenge, and I didn't go after those this round. That's a good next rep to come back to on my own, specifically to practice finding evidence with no step-by-step guide at all, since the whole point of a real case is that nobody hands you the filenames in advance.

I'd also want practice scripting the signature check itself. Doing it by hand in Autopsy's GUI, click a file, check ASCII Strings, check Hex, is fine for five files in a lab, but a real drive image can have thousands of files worth checking. A script that walks a file listing and flags anything whose actual signature doesn't match its extension would catch exactly what this lab manually walked through, faster and at scale.

## Why this matters for the job

Every evasion trick in this lab, wrong extension, deletion, burying a file inside an archive, is something a real suspect or malicious insider actually does, and every one of them fails against the same simple check: the file's real signature versus its claimed name. That's the core skill this lab is really testing, not clicking through Autopsy's menus. Combining that with a raw keyword search that doesn't depend on the filesystem's own directory structure being intact gives a forensic analyst a way to find evidence even when someone has actively tried to make the filesystem itself lie about what's there.
