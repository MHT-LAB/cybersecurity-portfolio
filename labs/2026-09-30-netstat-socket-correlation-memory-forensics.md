# Socket-to-Process Investigation: Finding, Dumping, and Killing a Rogue Listener on Kali

**Domain:** Security Operations / Incident Response
**Tools used:** netstat, ps, gcore (GDB), strings, grep, kill, python3
**Type:** Self-directed hands-on lab on a Kali Linux VM (CompTIA CySA+ prep, WGU D483 Security Operations)

## Scenario
I wanted to practice the core triage loop an analyst runs on a suspicious Linux host: find an unexpected network listener, tie it to a process, capture that process's memory before killing it, and confirm the threat is gone. To get something realistic to hunt, I started a throwaway Python web server on port 8080 in my Kali VM and treated it like an unauthorized listener I'd just been alerted to.

## Approach

**Phase 1: Learning what netstat actually shows me on Linux**
1. I started with `netstat -ano` out of Windows habit. The output was mostly Unix domain sockets (local IPC like the D-Bus and journald paths) plus one UDP entry, which was the VM's DHCP exchange with the local gateway. Nothing in it was useful for finding a server.
2. The bigger lesson was that `-o` doesn't mean the same thing across platforms. On Windows it shows the owning PID. On Linux it shows network timers (the `off (0.00/0/0)` values). On Linux, getting the PID takes `-p`, and it needs `sudo` to show processes I don't own.
3. I switched to `sudo netstat -tulpn`: TCP and UDP only, listening sockets only, with PID/program name and numeric addresses so there are no DNS lookups slowing things down or hiding the real port.

**Phase 2: Simulating the threat and detecting it**
1. I launched `python3 -m http.server 8080 &` to act as the rogue service.
2. Re-running `sudo netstat -tulpn` showed a new TCP listener on `0.0.0.0:8080` in the LISTEN state, owned by `python3`. Binding to `0.0.0.0` matters because it means the service is reachable on every interface, not just localhost.
3. I verified the process behind that PID with `ps aux` before touching it, so I wasn't acting on the socket table alone.

**Phase 3: Memory capture, artifact extraction, and eradication**
1. `sudo gcore -o http_server.dump <PID>` failed with `command not found`. `gcore` ships with the GNU Debugger, so I installed it with `sudo apt update && sudo apt install -y gdb` and re-ran it.
2. The dump was written as `http_server.dump.26150`. `gcore` appends the PID to the output name on its own, which matters for the next step.
3. I pulled printable strings and filtered for HTTP-related indicators: `strings http_server.dump.26150 | grep -E "GET|POST|HTTP" | head -n 10`.
4. I killed the process with `sudo kill -9 26150` (SIGKILL) and re-ran `sudo netstat -tulpn` to confirm 8080 no longer appeared.

**Mistakes I hit along the way (and what they taught me)**
- `sudo kill -9 python3` failed with a parse error. `kill` only accepts numeric PIDs. If I want to terminate by name, that's `killall` or `pkill`, which is also riskier because it can hit more than one process.
- `strings http_server.dump 26150` failed with "No such file." Two things went wrong at once: I forgot `gcore` adds the PID to the filename, and the stray space made `strings` treat `26150` as a second filename.

## Findings

| Item | Result |
|---|---|
| Suspicious listener | `python3` on `0.0.0.0:8080/tcp`, LISTEN |
| Exposure | Bound to all interfaces, so reachable from the network |
| Memory dump | `http_server.dump.26150` created with `gcore` |
| Strings output | Server startup banner ("Serving HTTP on 0.0.0.0 port 8080"), standard Python HTTP/RFC reference text, and initialization headers |
| Eradication | `kill -9` on the PID; port 8080 gone from the socket table afterward |

The strings output was thin, which makes sense: the server had received no real requests, so there was no attacker traffic in memory to recover. If this had been a real compromise, I'd expect to find request paths, client IPs, injected commands, or loaded modules in the dump, and I'd record those as IoCs. In a ticket I'd write up the process name, PID, bound address and port, parent process, start time, and the hash of the dump file.

Workflow I'd document for a repeatable runbook: socket → PID → process verification (`ps aux`) → memory dump (`gcore`) → string/IoC extraction → termination → socket re-check.

## What I'd do differently / lessons learned
- Capture more context before killing anything. In a real incident I'd also grab `ps` details (parent PID, command line, start time), the process's open files, and the binary path, since SIGKILL destroys that evidence and kills any chance of looking at it live.
- Hash the memory dump and store it somewhere read-only to preserve chain of custody.
- Use `ss -tulpn` going forward. `netstat` is deprecated on modern Linux and `ss` is the current tool, so I should be fluent in both because I'll still see `netstat` in older environments and exam questions.
- Check the dump with more than a single `grep` for HTTP keywords. Wider string searches, plus looking at the process's network-related strings, would catch more.
- Start with a graceful `SIGTERM` when evidence preservation and clean shutdown matter, and reserve `kill -9` for processes that won't respond.
- Re-practice the filename/spacing details in `gcore` and `strings` so I stop burning time on avoidable typos.

## Why this matters for the job
Tying a suspicious port to the exact process behind it, and preserving that process's memory before containment, is bread-and-butter work for a SOC analyst or incident responder. Knowing the platform differences (like `-o` on Windows vs. Linux) keeps me from misreading output or wasting time during a live investigation.
