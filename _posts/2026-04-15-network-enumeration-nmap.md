---
title: "Nmap — Host Enumeration | Hack The Box (HTB)"
date: 2026-04-15 00:00:00 +0530
permalink: /network-enumeration-nmap/
categories: [Enumeration, Tools]
tags: [nmap, network, htb, pentesting, enumeration, host-discovery]
---

![Nmap Banner](/assets/img/posts/nmap.jpg)

Network Mapper (Nmap) is a tool used for enumeration — one of the most critical parts of this field. It sends raw packets at a target and listens to what comes back. From that alone it can tell you what hosts are alive, what ports are open, what services are running, and sometimes even what OS you're dealing with. It's the first tool in almost every pen tester's workflow for a reason.

---

## Enumeration

Enumeration is the most critical part of all. You can't break things you don't understand or aren't familiar with — you should always gather as much information as possible from the service you are targeting. The more information you have, the easier it is to take over the system.

> **"Enumeration is the key."**

---

## Nmap

Nmap can be divided into the following scanning techniques:

- **Host discovery** — tells whether a host is up
- **Port scanning** — scans which ports are open
- **Service enumeration and detection** — tells what services are running on the host
- **OS detection** — detects the OS on which the host is running
- **Nmap Scripting Engine (NSE)** — for advanced enumeration and interaction with services

In this post, I want to share my learning from the **Host Discovery** module.

**Syntax:**
```bash
nmap <scan types> <options> <target>
```

---

## Host Discovery

Knowing which systems are online is an important first step when pentesting a network.

We can find live hosts in many ways:

---

### 1. Scan a Network Range

```bash
sudo nmap 10.129.2.0/24 -sn -oA tnet | grep for | cut -d" " -f5
```

| Flag | What it does |
|---|---|
| `10.129.2.0/24` | Target network range |
| `-sn` | Disables port scanning |
| `-oA tnet` | Stores results in all formats with filename 'tnet' |

Here we are using ICMP echo requests — a network packet sent to a destination to test connectivity and latency.

> **Note:** Fewer results doesn't mean those are the only hosts up. Cross-check the results — Nmap marks hosts as inactive when it receives no response, but firewalls may be dropping packets silently.

---

### 2. Scan from an IP List

```bash
sudo nmap -sn -oA tnet -iL hosts.lst | grep for | cut -d" " -f5
```

| Flag | What it does |
|---|---|
| `-iL` | Performs scans against targets listed in `hosts.lst` |

---

### 3. Scan a Single IP

```bash
sudo nmap 10.129.2.18 -sn -oA host
```

Even though we are sending an ICMP echo request, Nmap actually sends an **ARP ping** on the same subnet, resulting in an ARP reply.

ARP is useful within the same subnet because it is faster, more reliable, and cannot be easily blocked by firewalls.

To see exactly what packets are being sent, use the `--packet-trace` flag:

```bash
sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace
```

| Flag | What it does |
|---|---|
| `--packet-trace` | Shows all packets sent and received |

![packet trace output](/assets/img/posts/packet_trace.png)

**Analysis:**
- `-PE` performs the ping scan using ICMP Echo requests against the target
- `10.10.14.2` — attacker IP sending the request, asking if `10.129.2.18` is in its subnet
- `10.129.2.18` — replies with its MAC address, confirming the host is alive

---

To see *why* Nmap marked the host as up, add the `--reason` flag:

```bash
sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace --reason
```

![reason output](/assets/img/posts/host_discovery_arp.png)

**Analysis:**
- `--reason` displays the reason for a specific result
- Output: `Host is up, received arp-response (0.028s latency)` — confirms the host is alive and reachable on the local network, with a 28ms response time

---

### Forcing ICMP Instead of ARP

To disable ARP and force ICMP, use `--disable-arp-ping`:

```bash
sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace --disable-arp-ping
```

![disable arp ping output](/assets/img/posts/icmp.png)

By disabling ARP, Nmap used ICMP instead. The reply confirms the host is alive, and **TTL=128** suggests the target may be running **Windows** (Linux typically returns TTL=64).

---

## Quick Reference

| Command | What it does |
|---|---|
| `nmap -sn 10.0.0.0/24` | Find alive hosts, no port scan |
| `nmap -iL hosts.lst` | Scan from a hosts file |
| `nmap -PE --disable-arp-ping` | Force ICMP host discovery |
| `nmap --packet-trace` | Show every packet sent/received |
| `nmap --reason` | Show why host is marked up/down |
| `nmap -oA output` | Save results in all formats |

---

*HTB Academy — CPTS module notes. All scans performed on dedicated lab machines.*
