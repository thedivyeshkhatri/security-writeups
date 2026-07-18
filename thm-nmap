# Nmap — TryHackMe

**Date:** 2026-07-19 
**Category:** Network, Tooling  
**Skills used:** Nmap (host discovery, port scanning, service/OS detection)  

> **TL;DR:** Worked through Nmap end to end — discovering live hosts, running basic and
> advanced port scans, and using post-scan detection to fingerprint services, versions, and
> operating systems. This is a foundational tooling module, so it doubles as my Nmap reference.

## Overview
Nmap (Network Mapper) is the standard open-source tool for network discovery and scanning. A
typical workflow moves through four stages: **find live hosts → scan their ports → use advanced
scans where needed → fingerprint what's running.** Each stage narrows the picture of the target.

## 1. Live Host Discovery
Before scanning ports, find out which hosts are actually up, so you don't waste time scanning
dead addresses.
- `-sn` — **ping scan / no port scan**: discover live hosts only.
- On a local network, Nmap uses **ARP requests** (fast and reliable); across networks it falls
  back on **ICMP echo**, plus TCP/UDP probes to common ports.
- Understanding this stage matters because firewalls often block ICMP, so "no response" doesn't
  always mean "host down."

## 2. Basic Port Scans
Once hosts are known, find which ports are open and what state they're in. Nmap reports ports as
**open**, **closed**, or **filtered** (a firewall is likely dropping the probe).
- `-sT` — **TCP Connect scan**: completes the full TCP handshake. Works without special
  privileges, but noisier and more likely to be logged.
- `-sS` — **SYN / "half-open" scan**: sends SYN, sees the response, never completes the
  handshake. Faster and stealthier; needs elevated privileges. This is the common default.
- `-p` controls which ports: `-p 80,443` (specific), `-p-` (all 65535), `-F` (fast / top 100).

## 3. Advanced Port Scans
Techniques for scanning UDP, evading simple filters, or probing unusual setups.
- `-sU` — **UDP scan**: many key services (DNS, SNMP, DHCP) run over UDP. Slower because UDP is
  connectionless and gives less feedback.
- **Null (`-sN`), FIN (`-sF`), and Xmas (`-sX`) scans** — send unusual flag combinations to try
  to slip past basic firewalls and infer port state from the response (or lack of one).
- Evasion options like decoys (`-D`), fragmentation (`-f`), and timing controls (`-T0`–`-T5`)
  adjust how noisy or stealthy a scan is.

## 4. Post-Port Scans (Detection & Fingerprinting)
Knowing a port is open isn't enough — you want to know *what's behind it*.
- `-sV` — **service/version detection**: identifies the software and version on each open port.
- `-O` — **OS detection**: fingerprints the target operating system.
- `-sC` — runs **default NSE scripts** (Nmap Scripting Engine) for extra enumeration; `-A`
  bundles `-sV`, `-O`, and `-sC` together.
- **Output formats** for saving results: `-oN` (normal), `-oX` (XML), `-oG` (greppable),
  `-oA` (all at once).

## The Defensive View
Recon like this is exactly what defenders watch for:
- **Detection:** port scans produce distinctive traffic patterns (many connection attempts
  across ports); IDS/IPS and log monitoring flag them.
- **Reducing exposure:** close unused ports and disable unneeded services, so a scan finds
  less. **Firewalls** move ports from *open* to *filtered*, denying attackers easy information.

## What I Learned
Nmap made the recon process concrete: you move from "which hosts exist" to "which ports are
open" to "what's running on them," and each stage informs the next. The most useful takeaways
were the difference between **`-sT` and `-sS`** (full vs half-open, and the stealth/privilege
trade-off) and how **`-sV` and `-O`** turn a bare list of open ports into an actual map of the
target's software and OS — which is what makes the results useful for everything downstream.
