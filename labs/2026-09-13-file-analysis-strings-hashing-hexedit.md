# File Analysis Fundamentals: Strings, Hash Verification, and Hex Editing

**Domain:** Digital Forensics
**Tools used:** Kali Linux, `strings`, `md5sum`, `unzip`, `hexeditor`
**Type:** Hands-on lab (WGU D483 Security Operations)

## Scenario

The goal of this lab was to practice the first moves an analyst makes when handed an unknown or suspicious file: figure out what it is and what it might do before doing anything riskier with it. I worked through three related techniques on a Kali VM — pulling printable strings out of different file types, verifying a downloaded forensic test image against its known hash, and using a hex editor to hunt down specific byte patterns inside a raw disk image. The idea was to build a toolkit for triaging a file with nothing more than command-line utilities, which is exactly the kind of first-pass analysis a SOC analyst or forensic examiner does before escalating or diving deeper.

## Approach

**1. Extracting strings from different file types**
I started in a directory of sample files (image formats, an executable, plain text) and ran `strings` against each one to see what fell out. A plain `.txt` file just echoed back its contents, minus formatting — not surprising, since the whole file is printable text. The interesting part was comparing image formats: an `.ico` file produced no output at all, meaning it had no runs of 4+ printable ASCII characters anywhere in it, while `.bmp`, `.gif`, and `.png` files all shared an identical embedded string from the software that had generated them. That told me metadata like "created with X" gets baked into image files as plain text even though the pixel data itself is binary — a useful reminder that "binary file" doesn't mean "no readable strings."

I then ran `strings` against a Windows executable and redirected the output to a file so I could page through it with `less`. Right away I recognized a lot of noise: standard C runtime strings like `malloc`, `memcpy`, `strlen`, and `fprintf` — none of that describes what the program does, it's just linker/compiler artifacts that show up in nearly every compiled binary. The real signal in a triage like this is picking out the strings that *aren't* boilerplate: version info, embedded messages, and any hardcoded paths or indicators.

**2. Verifying file integrity with a hash check**
Next I pulled a known digital forensics test image (a standard EXT3 filesystem test file used for practicing forensic tool testing) from local removable media rather than downloading it directly, since the lab environment had no live internet access. After extracting it from its zip archive, I generated an MD5 hash of the file with `md5sum` and compared it against the hash published by the source site. They matched, which confirmed the copy I had was bit-for-bit identical to the original — no corruption in transit, no tampering.

While doing this I worked out a quick mental trick for identifying an unlabeled hash by its length: count the hex characters and multiply by 4 to get the bit length (32 hex chars = 128 bits = MD5, 64 hex chars = 256 bits = SHA-256, and so on). That's a fast sanity check when you're handed a hash with no algorithm specified.

**3. Locating known strings with a hex editor**
With a set of strings already identified from the `strings` output of the disk image, I opened the same file in `hexeditor` and used it to search for those specific terms rather than scrolling manually — on a 5MB raw disk image, blind paging is a losing strategy, but a targeted search using CTRL+W jumps straight to a byte offset. I logged the hex offsets where specific strings appeared, tested what happens when a search term *doesn't* exist in the file (a clean "not found" rather than a false positive), and confirmed I had to reset the cursor to offset 0x00000000 before starting each new search, or it would only search forward from the current position.

## Findings

- Image file metadata (BMP/GIF/PNG in this case) commonly retains an identical creator/tool string across formats when produced by the same software — a low-effort artifact to check when trying to fingerprint the origin of a file.
- An `.ico` file with zero string output isn't necessarily suspicious on its own, but it's a good example of why `strings` alone can give a false sense of "nothing here" — the actual content still needs a hex-level look if something else about the file is off.
- Executable strings output is dominated by compiler/runtime noise (`malloc`, `strlen`, `fprintf`, DOS-stub messages like the classic "This program cannot be run in DOS mode"). An analyst has to filter that out to find anything meaningful, which is exactly why automated string-matching against a "known-noise" list would save real time in a SOC.
- Hash verification (MD5 in this case) confirmed integrity of the test image end-to-end. Worth flagging in any real report: MD5 is deprecated for security purposes due to collision vulnerabilities, even though it's still fine for basic integrity checks against a known-good hash from a trusted source. SHA-256 or SHA-512 would be the better choice if collision resistance actually mattered.
- Hex-level search confirmed multiple recurring instances of the same string at different offsets within the raw disk image — consistent with a filesystem that had that keyword written into it more than once (e.g., across multiple files or file system structures within the image).

## What I'd do differently / lessons learned

Manually paging through `less` output and eyeballing strings works fine for a handful of files, but it doesn't scale. In a real environment I'd want to pipe `strings` output through `grep` for known indicators of compromise, or run it against a baseline "known good" string list to auto-filter compiler noise. For the hex editor portion, doing repeated manual searches and having to reset the offset each time was tedious — a scripted approach (e.g., `grep -abo` to get byte offsets of a string directly) would get the same answer faster than driving `hexeditor` interactively. I'd also want more practice with hex editors generally; right now I'm still translating hex offsets in my head rather than reading them fluently, and that's a skill that only comes from repetition.

## Why this matters for the job

This is the exact workflow a SOC or DFIR analyst runs before escalating an unknown file: cheap, non-destructive checks first (strings, hashing) to build context, then a more surgical hex-level examination once you know what you're looking for. Being fast and comfortable with `strings`, `md5sum`/`sha256sum`, and a hex editor means less time waiting on a sandbox or full malware analysis pipeline for the easy calls.
