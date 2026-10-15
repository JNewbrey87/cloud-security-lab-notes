# Volatile Memory Capture and Analysis: Live Acquisition from a Windows Jump Box

**Project:** CySA+ Lab Track, Digital Forensics and Incident Response
**Session:** 02 | **Platform:** VirtualBox on Windows 11 Pro, isolated host-only network
**Acquisition host:** LAB-WSJP-25 (Windows Server 2025, Tier 1 jump box) | **Analysis host:** LAB-KALI
**Tools:** WinPmem v4.0.rc1, Volatility3 2.28.2
**MITRE ATT&CK (detection relevance):** [T1059.001 — PowerShell](https://attack.mitre.org/techniques/T1059/001/), [T1218 — System Binary Proxy Execution (LOLBins)](https://attack.mitre.org/techniques/T1218/)
**Cert Alignment:** CySA+ CS0-004, Domain 4 (Incident Response and Digital Forensics): order of volatility, live acquisition, chain of custody, memory artifact triage
**Date:** October 2026

---

## Scenario

Volatile memory is the most perishable evidence on a running system. RFC 3227's order of volatility puts RAM near the top, above disk and well above backups, because it holds state that vanishes the moment the machine reboots or loses power: running processes, injected code, decrypted material, live network connections, and command lines that were never written to disk. Wait, and it is gone. This lab runs the full preservation workflow against a live host: acquire physical memory to an isolated evidence destination, hash it at the source, verify the hash on arrival, lock the original read-only, then triage it with Volatility3.

The subject is the Tier 1 jump box from my existing lab, the same host that brokers administrative sessions in the kill-chain and vulnerability-management work. Privileged credentials pass through a jump box, which makes it both a realistic target and a realistic thing to have to image under pressure.

## Preparing the Evidence Destination

The single most important forensic decision here is where the image lands. Writing a memory dump to the subject's own disk alters the subject and risks overwriting the very evidence you are trying to preserve. So the dump goes to an isolated SMB evidence share on the Kali analysis host instead, and the acquisition writes to it directly over the host-only segment.

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

`map to guest = never` is load-bearing: without it, Windows Server 2025 rejects the connection with error 1272 (guest-access policy block). The share is scoped to a single account, not opened to Everyone. One practical prerequisite worth noting for future large-artifact labs: the Kali VDI had to be grown from 40 GB to 60 GB first, because a resident Nessus install left too little headroom for an 8.3 GB image.

## Acquisition with Chain of Custody

A pre-acquisition UTC timestamp was recorded, then WinPmem wrote the image straight to the mapped evidence share:

```powershell
# Administrator PowerShell on LAB-WSJP-25
C:\temp\winpmem.exe Z:\memdump_20261015.raw
```

WinPmem extracts a signed kernel driver to a temp path, loads it, maps physical memory, streams it to the target, then unloads and deletes the driver, leaving no persistent system change. The image was 8,925,478,912 bytes, consistent with the VM's 8 GB RAM allocation.

Chain of custody is the part that turns a memory dump into evidence:

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

The hash was taken on the source host immediately after acquisition and recomputed on Kali after transfer; the two matched, and the image was then marked read-only to stand in for a hardware write-blocker. A capture you cannot prove is unchanged is not evidence, it is just a file.

### Acquisition Footprint

Live acquisition necessarily touches the system it acquires, and the honest move is to document that rather than pretend otherwise:

| Artifact | Detail | Persistent? |
|----------|--------|-------------|
| Process entry | `winpmem.exe` (PID 8992), child of `powershell.exe` | No, exits after acquisition |
| Kernel driver | Extracted to `%TEMP%\pme####.tmp`, loaded, then unloaded and deleted | No, removed at completion |
| Local disk write | None; the dump went to the network share | N/A |
| Network connection | Outbound SMB (445) to the evidence share | No, mapping removed after |
| Memory modification | WinPmem's own process and driver data appear in the dump | Expected, the act of acquisition is recorded |

This is inherent to live memory forensics and consistent with RFC 3227. The dump captures its own acquisition; you document it and move on.

## Analysis with Volatility3

### windows.pslist

The process list proves the capture is genuinely live, because the acquisition tool captured its own process entry. The chain `explorer.exe (4924) → powershell.exe (7648) → winpmem.exe (8992)` is fully recoverable, which is exactly the provenance a reviewer should be able to confirm from the image itself.

| PID | PPID | Process | Note |
|-----|------|---------|------|
| 4924 | 984 | explorer.exe | Desktop shell |
| 7648 | 4924 | powershell.exe | The admin session |
| 8992 | 7648 | winpmem.exe | The acquisition tool |
| 6900 | 4924 | shutdown.exe | Resolved below, with evidence |
| 6816 | 4924 | AzureArcSysTra | Azure Arc agent, see Cloud Forensics |
| 3084 | 4924 | VBoxTray.exe | Confirms the VirtualBox guest |

**`shutdown.exe` (PID 6900), resolved with evidence, not a guess.** A shutdown binary sitting in memory earns immediate attention, because adversaries trigger reboots to flush exactly the volatile evidence this capture preserves. Rather than speculate, I queried the Windows System log for Event ID 1074 (shutdown initiated):

```powershell
Get-WinEvent -LogName System | Where-Object {$_.Id -eq 1074} |
  Select-Object -First 5 TimeCreated, Message | Format-List
```

Every recent 1074 event was from 2026-10-13, all attributed to `TrustedInstaller.exe` and `MoNotificationUx.exe` ("Operating System: Upgrade (Planned)"), which match the October 2026 cumulative updates installed that day. There was no 1074 event for the acquisition date at all. The evidence-based conclusion: `shutdown.exe` was a residual process from the prior day's update cycle, not an adversarial reboot, and it never completed a shutdown. Flag first, then resolve from the record; the difference between those two sentences is the difference between a finding and a hunch.

### windows.cmdline

PowerShell ran from its standard System32 path, and the WinPmem command line matched what was executed. No living-off-the-land binaries (wscript, mshta, rundll32, cscript) appeared. This is the memory-resident version of the LOLBin hunt that maps to T1059 and T1218: command lines of processes that have since exited are still in RAM, available here even when nothing was logged to disk.

### windows.netscan versus windows.netstat

`windows.netscan` returned no connections, and the interesting part is why. Verbose output confirmed Volatility3 did load a valid symbol table for this kernel (`ntkrnlmp.pdb/A90C614A...`); the repeated "Symbol table requirement not yet fulfilled" lines are the automagic framework iterating through layer permutations, which is normal, not an error. The real failure is at the plugin level: netscan parses `_TCP_ENDPOINT` and `_TCP_LISTENER` structures whose layout changed in Windows Server 2025 (Build 26100) in a way that this Volatility build's netscan parser does not account for, so it finds nothing to parse.

That correction matters, because the obvious-sounding fix is wrong. A missing symbol table was not the cause, and `dwarf2json` is not the remedy; `dwarf2json` builds Volatility symbol tables from Linux and macOS DWARF data and has no bearing on Windows kernel symbols (those come from PDBs via `pdbconv.py`), and even a freshly built table would not fix a structure-parsing gap inside the plugin. The right move is to try a different plugin or a newer Volatility.

`windows.netstat` uses a different traversal path, and it recovered the network state in full:

| Proto | Local | Port | State | PID | Owner |
|-------|-------|------|-------|-----|-------|
| TCPv4 | 0.0.0.0 | 445 | LISTENING | 4 | System |
| TCPv4 | 0.0.0.0 | 3389 | LISTENING | 1260 | svchost.exe |
| TCPv4 | 0.0.0.0 | 5985 | LISTENING | 4 | System (WinRM) |
| TCPv4 | 192.168.10.60 | 139 | LISTENING | 4 | System (host-only NIC) |
| TCPv4 | 10.0.2.15 | 139 | LISTENING | 4 | System (NAT NIC) |
| TCPv4 | 127.0.0.1 | 40341/40342 | LISTENING | 7124 | himds.exe (Arc) |
| TCPv4 | 127.0.0.1 | 40343/40344 | LISTENING | 3796 | arcproxy.exe (Arc) |

Two bound IPs confirm a dual-homed host: `192.168.10.60` on the lab segment and `10.0.2.15` on NAT. A dual-homed jump box is a pivot point, and in a lateral-movement investigation both interfaces and their owning processes have to be mapped. The lesson to bank: on Server 2025, run both netscan and netstat and document which one answered.

## The Cloud Forensics Bridge

The most consequential artifact in the capture is Azure Arc, and this run confirmed it from two independent planes. The process list shows `AzureArcSysTra`; the network state shows `himds.exe` (the Arc Hybrid Instance Metadata Service) and `arcproxy.exe` holding local listener ports. Two different data structures, same conclusion: this on-premises host has live telemetry channels into Azure.

That is what makes a memory-forensics exercise a cloud security lab. Arc enrollment means cloud-side forensic artifacts exist for a machine that physically lives in the lab, so discovering it on a host widens the investigation: the IR scope now includes the Azure tenant, where Arc agent activity logs, Defender for Cloud alerts, and forwarded telemetry all become evidence. The memory image is what tells you the cloud side even exists. A follow-on session will pull the Azure-side forensic snapshot for this same host and reconcile it against what the local capture showed.

## What I Took Away

**Chain of custody held across a network transfer.** The SHA-256 matched between source and destination, the dump never touched the subject's disk, and the original was locked read-only. That is the memory-forensics equivalent of a clean evidence handoff, and it is the first thing a reviewer should be able to verify.

**Evidence beats speculation, and the log was right there.** The earlier instinct on `shutdown.exe` was a reasonable guess; Event ID 1074 turned it into a documented conclusion. The habit of resolving a flag from the record instead of from intuition is most of what separates triage from storytelling.

**Netscan and netstat are not interchangeable.** They read different kernel structures, and on Server 2025 one failed while the other succeeded. A negative result you can explain (a plugin-level parsing gap, not a missing symbol table) is a finding; a negative result you cannot explain is a liability.

**Arc enrollment is a scope decision.** The cloud-security takeaway is not a command, it is recognizing that one agent, confirmed across the process and network planes, redraws the boundary of an investigation. That intersection of digital forensics and cloud is the work I am building toward.

## Lab Environment

| Component | Details |
|-----------|---------|
| Acquisition target | LAB-WSJP-25, Windows Server 2025 (Build 26100), Tier 1 jump box, 8 GB RAM |
| Acquisition tool | WinPmem v4.0.rc1 (signed driver, non-destructive) |
| Analysis host | LAB-KALI, Volatility3 2.28.2; isolated SMB evidence share |
| Evidence handling | Written to network share, hashed at source and verified on arrival, original `chmod 444` |
| Network | Isolated host-only VirtualBox network (192.168.10.0/24), no external exposure |

## Follow-On Items

- Upgrade Volatility3 to a build with Server 2025 netscan support, then re-run and compare against the netstat results.
- Run `windows.dlllist` against the PowerShell process for module-level triage.
- Pull the Azure-side forensic snapshot for this host and reconcile it with the memory capture; review the Arc workspace ID and log retention as the prerequisite for that cloud forensic session.

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and the CySA+ / SC-200 / SC-500 certifications.*
