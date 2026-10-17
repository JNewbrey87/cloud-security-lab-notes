# Cloud Forensic Acquisition: Snapshotting a Running Azure VM

**Project:** CySA+ Lab Track, Incident Response  
**Session:** 04 | **Platform:** Microsoft Azure, Azure CLI in Cloud Shell (Bash)  
**Target:** Windows Server 2022 Datacenter VM, Standard_D2s_v3, East US  
**Cert Alignment:** CySA+ CS0-004, Domain 3 (Incident Response and Management), with Domain 1 (Security Operations) as secondary  
**Date:** October 2026  

---

## Why I Did This One

In cysa-02, I captured memory from one of my on-prem lab VMs and found the Azure Arc agent running on it. That got me thinking about the cloud side of forensics. Arc gives Azure a management and telemetry view of that box, but the VM itself lives in my homelab, so there's no Azure disk to snapshot.

Standing up VMs is normal sysadmin work for me. What I hadn't done was treat a cloud VM as evidence. So for this one, I wanted to teach myself how acquisition works when the machine actually lives in Azure: spin up a Windows Server VM, snapshot its OS disk while it was still running, write down the chain-of-custody details, and tear it all down before it could keep billing me.

## Setting Up the VM

I wanted the free-tier B1s size, but East US showed it as "Size not available" when I went to provision. I went with Standard_D2s_v3 (2 vCPU, 8 GiB) instead.

That size isn't free tier, so this lab ran on metered pay-as-you-go billing at $0.1880/hr, and the whole session came in under $0.20. I keep my lab work free-tier only, so this was a deliberate exception, and I'd rather say so than pretend it wasn't.

Two things almost got past me in the portal. First, the image picker defaulted to Windows Server 2025 Datacenter Azure Edition, which carries extra licensing cost; I switched it to plain Windows Server 2022 Datacenter. Second, the options to delete the OS disk, NIC, and public IP along with the VM were all unchecked by default. If I'd deleted only the VM, those pieces would have stuck around and kept costing money. That's part of why I tore the lab down at the resource group level instead of resource by resource.

I left RDP (3389) open so I could reach the box during the lab. That's a lab shortcut. In production I'd keep it closed and use something like Bastion or just-in-time access instead.

## Taking the Snapshot

This is the part I really wanted to understand. I took the snapshot from Cloud Shell while the VM was still running:

```bash
az snapshot create \
  --resource-group rg-security-lab \
  --name snap-forensic-20261017 \
  --source VM-Forensic-LAB_OsDisk_1_ba30f93219fe4cb5b680883ed3c542c3 \
  --location eastus
```

It came back `provisioningState: Succeeded`, created at 2026-10-17 16:45:45 UTC.

What clicked for me is that this happens at the Azure control plane, against the managed disk resource. I never RDP'd in, never installed an agent, never touched the guest OS. It's about the closest cloud equivalent I can think of to imaging a physical drive behind a write blocker, because the acquisition happens below the operating system.

Snapshotting a running VM gives you a crash-consistent copy: the disk as it would look if someone pulled the power. Anything still in memory isn't there, which is exactly why cysa-02 was a separate lab. For disk artifacts like the file system and registry hives, crash-consistent is generally what you'd want for this kind of acquisition anyway.

## Chain of Custody

I hashed the snapshot's identifying metadata into a single SHA-256 so I'd have a tamper-evident record of exactly which snapshot I took:

```bash
echo -n "snap-forensic-20261017|2026-10-17T16:45:45.8804439+00:00|01d26e1a-063c-4950-ab47-d3de7af3edb6|ba30f932-19fe-4cb5-b680-883ed3c542c3|127GB|Windows|eastus" | sha256sum
```

`4b044a8c3cef2da3f61f7108ca31a7c1187354cb09d57e457d8549146ba1dade`

I want to be upfront about what that hash does and doesn't prove. It covers the snapshot's metadata (name, creation time, unique ID, source disk ID, size, OS type, region), not the actual disk bytes. It shows this specific snapshot hasn't been renamed or re-pointed at a different source. It does not prove the data on the disk is unchanged. A real disk-byte hash would mean exporting the snapshot and hashing the VHD itself, which comes with storage and transfer costs I wasn't willing to take on under my free-tier rule. So for this lab it's a metadata integrity record, not an image hash, and I'm calling it that.

| Field | Value |
|-------|-------|
| Snapshot name | snap-forensic-20261017 |
| Snapshot unique ID | 01d26e1a-063c-4950-ab47-d3de7af3edb6 |
| Source disk unique ID | ba30f932-19fe-4cb5-b680-883ed3c542c3 |
| Created (UTC) | 2026-10-17 16:45:45 |
| Size | 127 GB (136,367,308,800 bytes) |
| Type | Full snapshot (not incremental), Standard_LRS |
| Encryption | Encryption at rest with a platform-managed key |
| VM profile | Hyper-V Gen 2, Trusted Launch (Secure Boot plus vTPM) |
| Metadata SHA-256 | `4b044a8c3cef2da3f61f7108ca31a7c1187354cb09d57e457d8549146ba1dade` |

## Tearing It Down

```bash
az group delete --name rg-security-lab --yes --no-wait
```

That queued deletion of everything in the group: the VM, OS disk, NIC, network security group, public IP, and the snapshot itself. I checked the portal afterward and confirmed it was all gone.

This is where a shortcut caught up with me. I was short on time, so instead of creating a fresh resource group, I built this VM in the one I set up months ago for my earlier Azure work. What I didn't think through was that the group still held the Arc registrations for three of my on-prem lab machines, plus the Microsoft Cloud Security Benchmark policy assignments I'd scoped to it. Deleting the group took all of that out along with the lab VM. It was a clean lesson in blast radius: a resource group delete removes everything in the group, not just the thing you were thinking about when you typed the command.

## What I Took Away

**You can acquire evidence without touching the box.** Taking the snapshot through the control plane means the guest OS never knows it happened. That's a different way of thinking than on-prem acquisition, where you're usually working from inside or next to the machine.

**Know what your hash actually covers.** It would have been easy to post a SHA-256 and let it look like a disk image hash. It isn't. Being clear about the gap between a metadata hash and a disk-byte hash felt more honest, and more useful to me later, than a cleaner-looking writeup.

**Default encryption doesn't mean you hold the key.** The snapshot was encrypted at rest, but with a key Microsoft manages. If an investigation needed strict key custody, I'd want a customer-managed key set up before acquisition. Same idea with Trusted Launch: a vTPM can matter if a volume is BitLocker-protected and you're trying to analyze it offline later.

**The portal defaults aren't on your side.** Between the Azure Edition image and the unchecked delete options, I nearly ended up paying for things I didn't mean to. Checking the Review + Create page before deploying is a habit I'm keeping.

**A teardown command is only as safe as the scope you point it at.** Resource-group-level teardown is still the right pattern, because it doesn't leave orphaned NICs and IPs behind. But it only works if the group holds nothing except the lab. The fix is simple: cloud lab VMs get their own resource group, separate from anything tied to my on-prem lab machines, and I check what's in a group before I delete it.

## Lab Environment

| Component | Details |
|-----------|---------|
| Platform | Microsoft Azure, East US |
| VM | Standard_D2s_v3 (2 vCPU, 8 GiB), Windows Server 2022 Datacenter |
| Acquisition | `az snapshot create` from Azure Cloud Shell, VM running |
| Integrity | SHA-256 over snapshot metadata (not disk bytes) |
| Teardown | Resource group delete (also removed earlier Arc registrations and policy assignments in the same group) |
| Cost | Metered pay-as-you-go, under $0.20; a named exception to my free-tier rule |

## Follow-On

- Re-run on a B1s VM in a region where it's available, to stay inside the free tier, and in its own dedicated resource group.
- Re-onboard my lab machines to Arc in a resource group of their own, if I want that telemetry back.
- Look at what a full disk-byte hash would actually cost (exporting the snapshot and hashing the VHD) before deciding whether it's worth doing.
- Try acquisition with a customer-managed key, after confirming Key Vault's per-operation charges fit the free-tier rule.

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and the CySA+ / SC-200 / SC-500 certifications.*
