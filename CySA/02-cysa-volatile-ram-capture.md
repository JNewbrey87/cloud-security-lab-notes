# Volatile Memory Capture and Analysis: Live Acquisition from a Windows Jump Box

**Project:** CySA+ Lab Track, Digital Forensics and Incident Response  
**Session:** 02 | **Platform:** VirtualBox on Windows 11 Pro, isolated host-only network  
**Acquisition host:** LAB-WSJP-25 (Windows Server 2025, Tier 1 jump box) | **Analysis host:** LAB-KALI  
**Tools:** WinPmem v4.0.rc1, Volatility3 2.28.2  
**MITRE ATT&CK (detection relevance):** [T1059.001 — PowerShell](https://attack.mitre.org/techniques/T1059/001/), [T1218 — System Binary Proxy Execution (LOLBins)](https://attack.mitre.org/techniques/T1218/)  
**Cert Alignment:** CySA+ CS0-004, Domain 3 (Incident Response and Management), with Domain 1 (Security Operations) as secondary  
**Date:** October 2026  

---

## Why I Did This One

My first lab in this track got me thinking about the order of volatility, and I realized I'd never actually preserved memory from a live Windows box. I know my way around a running server, but capturing its RAM as evidence, in a way I could prove hadn't been changed, was new territory. RAM is about as perishable as evidence gets: running processes, network connections, and command lines that never hit the disk are gone the moment the machine reboots.

So I picked the jump box from my lab, the same machine that handles admin sessions in my kill-chain and scanning labs, and taught myself the whole workflow. Capture the memory, send it somewhere other than the machine itself, hash it on both ends, lock the copy read-only, and then dig into it with Volatility3.

I actually ran this lab twice. My first pass on October 14 worked, but when I reviewed it I found real problems: I hadn't hashed the image at acquisition, I'd written the dump to the jump box's own disk, and I'd misdiagnosed one of the plugin failures. So I went back on October 15 and redid the acquisition properly. This writeup is the corrected version.

## Setting Up a Place for the Evidence to Land

The first fix was where the image goes. Writing a memory dump to the machine you're investigating changes that machine and can overwrite the evidence you're after, which is exactly what I'd done on my first pass. This time I set up an SMB share on my Kali analysis box and had the acquisition write straight to it over the lab network.

```ini
# /etc/samba/smb.conf (relevant sections), on LAB-KALI
[global]
   map to guest = never
   security = user

[evidence]
   path = /home/kali/evidence/wsjp25-20261015
   valid users = kali
   guest ok = no
   writable = yes
```

The `map to guest = never` line took me a minute to figure out. Without it, Windows Server 2025 refused the connection with error 1272, which is its guest-access policy block. I also locked the share to a single account instead of opening it up to everyone. One more practical thing: I had to grow the Kali VM's disk from 40 GB to 60 GB first, because Nessus was already taking up enough space that an 8.3 GB image wouldn't fit.

## Capturing the Memory

I recorded a UTC timestamp before starting, then had WinPmem write the image directly to the mapped share:

```powershell
# Administrator PowerShell on LAB-WSJP-25
C:\temp\winpmem.exe Z:\memdump_20261015.raw
```

WinPmem drops a signed kernel driver into a temp folder, loads it, reads physical memory, streams it to the destination, then unloads and deletes the driver. The image came out to 8,925,478,912 bytes, which matches the VM's 8 GB of RAM.

### Chain of Custody

This is the part I skipped the first time and wanted to get right:

| Field | Value |
|-------|-------|
| File | memdump_20261015.raw |
| Source | LAB-WSJP-25 (192.168.10.60) |
| Destination | LAB-KALI evidence share (never the subject's disk) |
| Acquisition window | 2026-10-15 14:37:22 to 15:02:30 UTC |
| Tool | WinPmem v4.0.rc1 |
| File size | 8,925,478,912 bytes |
| SHA-256 at source | `B4EEC617661870D41A4A9227CB9DE0AF2A54E031D5F05D649466165BB90CB570` |
| SHA-256 on arrival | matched exactly |
| Integrity | PASS |
| Original after verification | read-only (`chmod 444`) |

I hashed the image on the jump box right after capture, hashed it again on Kali after the transfer, and the two matched. Then I set the file to read-only as my stand-in for a hardware write blocker. Without that hash match, I'd just have a big file and my word that it hadn't changed.

### What the Capture Left Behind

Capturing memory from a live machine changes that machine a little, no way around it. I wanted to write down exactly how, rather than call it "non-destructive":

| Artifact | Detail | Persistent? |
|----------|--------|-------------|
| Process entry | `winpmem.exe`, running as a child of `powershell.exe` | No, exits after acquisition |
| Kernel driver | Extracted to `%TEMP%\pme####.tmp`, loaded, then unloaded and deleted | No, removed at completion |
| Local disk write | None; the dump went to the network share | N/A |
| Network connection | Outbound SMB (445) to the evidence share | No, mapping removed after |
| Memory changes | WinPmem's own process and driver show up in the dump | Expected; the capture records itself |

So the footprint is minimal and documented. The dump includes the capture itself, which is just how live memory forensics works.

## Digging In with Volatility3

One note on sources before the results: the process list and command-line findings below are from my first capture on October 14. My notes from the redo recorded the process state as equivalent, so I kept the first run's details. The network results further down come from the October 15 image, the one with the verified hash.

### windows.pslist

The process list showed the capture tool in its own snapshot, which is how I knew this was a genuinely live capture. The parent chain was all there: `explorer.exe (4924) → powershell.exe (7648) → winpmem.exe (8992)`. That's a nice thing to have when you're documenting how evidence was collected, because the image itself shows where it came from.

| PID | PPID | Process | Note |
|-----|------|---------|------|
| 4924 | 984 | explorer.exe | Desktop shell |
| 7648 | 4924 | powershell.exe | The admin session |
| 8992 | 7648 | winpmem.exe | The capture tool |
| 6900 | 4924 | shutdown.exe | Looked into below |
| 6816 | 4924 | AzureArcSysTra | Azure Arc agent, see below |
| 3084 | 4924 | VBoxTray.exe | Confirms it's a VirtualBox guest |

**The `shutdown.exe` question.** Seeing a shutdown binary in memory made me stop, because attackers sometimes force a reboot specifically to wipe the kind of evidence I was trying to save. On my first pass I just guessed it was left over from patching. This time I checked the Windows System log for Event ID 1074, which records when a shutdown is started:

```powershell
Get-WinEvent -LogName System | Where-Object {$_.Id -eq 1074} |
  Select-Object -First 5 TimeCreated, Message | Format-List
```

Every recent 1074 event was from October 13, all from `TrustedInstaller.exe` and `MoNotificationUx.exe` ("Operating System: Upgrade (Planned)"), which lines up with the cumulative updates I installed that day. There was nothing at all on the day of the capture. So `shutdown.exe` was left over from the previous day's updates, it never actually shut anything down, and now I had the log to back that up instead of a guess.

### windows.cmdline

PowerShell was running from its normal System32 path, and the WinPmem command line matched what I typed. No living-off-the-land binaries showed up (wscript, mshta, rundll32, cscript). What I liked about this plugin is that command lines for processes that have already exited can still be sitting in RAM, even if nothing was logged to disk.

### windows.netscan, and Why netstat Saved Me

`windows.netscan` came back with nothing. On my first pass I blamed a missing symbol table and figured `dwarf2json` was the fix. Both were wrong. When I ran it verbose, Volatility3 had loaded a valid symbol table for this kernel just fine (`ntkrnlmp.pdb/A90C614A...`); the "Symbol table requirement not yet fulfilled" messages I'd been worried about are just the framework trying different layer combinations, which is normal. And `dwarf2json` builds symbol tables from Linux and macOS debug data, so it was never going to help with a Windows kernel anyway. Windows symbols come from PDB files through `pdbconv.py`.

The real problem was inside the plugin. netscan reads the `_TCP_ENDPOINT` and `_TCP_LISTENER` structures, and their layout changed in Windows Server 2025 (Build 26100) in a way this version of Volatility doesn't handle. So it ran, found nothing it could parse, and returned empty. A new symbol table wouldn't have fixed that.

`windows.netstat` walks a different path, and it got me the full network picture:

| Proto | Local | Port | State | PID | Owner |
|-------|-------|------|-------|-----|-------|
| TCPv4 | 0.0.0.0 | 445 | LISTENING | 4 | System |
| TCPv4 | 0.0.0.0 | 3389 | LISTENING | 1260 | svchost.exe |
| TCPv4 | 0.0.0.0 | 5985 | LISTENING | 4 | System (WinRM) |
| TCPv4 | 192.168.10.60 | 139 | LISTENING | 4 | System (host-only NIC) |
| TCPv4 | 10.0.2.15 | 139 | LISTENING | 4 | System (NAT NIC) |
| TCPv4 | 127.0.0.1 | 40341/40342 | LISTENING | 7124 | himds.exe (Arc) |
| TCPv4 | 127.0.0.1 | 40343/40344 | LISTENING | 3796 | arcproxy.exe (Arc) |

Two IPs means the jump box has two network interfaces: `192.168.10.60` on the lab network and `10.0.2.15` on NAT. A jump box sitting on two networks is a natural pivot point, so in a real lateral-movement investigation I'd want both interfaces mapped. My takeaway on the tooling: on Server 2025, run both netscan and netstat and write down which one worked.

## The Cloud Angle

The finding I found most interesting was Azure Arc, and it showed up two separate ways. The process list had `AzureArcSysTra`, and netstat showed `himds.exe` (Arc's Hybrid Instance Metadata Service) and `arcproxy.exe` listening locally. Two different places in memory, same answer: this on-prem machine was sending telemetry to Azure.

That changed how I thought about the investigation. If a machine is Arc-enrolled, there's evidence for it in Azure too, like Arc agent activity logs, Defender for Cloud alerts, and whatever telemetry got forwarded. So the scope of an investigation doesn't stop at the box itself, and the memory image is what told me the cloud side existed at all.

Arc gives Azure a management and telemetry view of this machine, but the VM itself still lives in my homelab, so there's no Azure disk to snapshot. That's what sent me to [cysa-04](04-cysa-azure-vm-forensic-snapshot.md), where I did a cloud disk acquisition on an actual Azure VM.

## What I Took Away

**The hash is what makes it evidence.** Matching SHA-256 on both ends, keeping the dump off the jump box's disk, and locking the copy read-only were the three things I skipped the first time. Going back and doing them is the biggest difference between my first pass and this one.

**Check the log before trusting the guess.** My first explanation for `shutdown.exe` turned out to be right, but I didn't actually know that until Event ID 1074 showed it. I'd rather have the record than a good hunch.

**Know why a tool came back empty.** netscan and netstat read different structures, and on Server 2025 one failed while the other worked. If I'd stopped at "no connections found," I would have published something wrong. Figuring out it was a plugin parsing problem, not a missing symbol table, also corrected a mistake from my own first pass.

**One agent can change the scope.** Seeing Arc in both the process list and the network state meant the investigation reached into Azure. That overlap between forensics and cloud is exactly where I want to keep building.

## Lab Environment

| Component | Details |
|-----------|---------|
| Acquisition target | LAB-WSJP-25, Windows Server 2025 (Build 26100), Tier 1 jump box, 8 GB RAM |
| Acquisition tool | WinPmem v4.0.rc1 (signed driver; minimal, documented footprint) |
| Analysis host | LAB-KALI, Volatility3 2.28.2; isolated SMB evidence share |
| Evidence handling | Written to network share, hashed at source and verified on arrival, original `chmod 444` |
| Network | Isolated host-only VirtualBox network (192.168.10.0/24), no external exposure |

## Follow-On

- Upgrade Volatility3 to a version that supports netscan on Server 2025, re-run it, and compare against the netstat results.
- Run `windows.dlllist` against the PowerShell process to practice module-level triage.
- Review Arc telemetry and log retention for this machine. That now needs a step first: the Arc registration for this host was removed when I tore down the resource group in cysa-04, so I'd have to re-onboard it to Arc before there's anything to review.

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and the CySA+ / SC-200 / SC-500 certifications.*
