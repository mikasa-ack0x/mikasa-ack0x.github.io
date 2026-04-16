---
title: "Nmap — Port Discovery | Hack The Box (HTB)"
date: 2026-04-16 00:00:00 +0530
categories: [Enumeration, Tools]
permalink: /posts/nmap-port-discovery/
tags: [nmap, network, htb, pentesting, enumeration, port-discovery]
---

> Part of my HTB CPTS journey. This post covers the fundamentals of port scanning: how Nmap discovers open services, interprets port states, and traces packets under the hood.

![Nmap Banner](/assets/img/posts/nmap.jpg)
---

## Why Port Scanning?

Once we confirm a target is alive, the next step is building a clearer picture of the system. We want to identify:

- Open ports and their services
- Service versions
- Information the services expose
- The underlying operating system

---

## The 6 Port States

Nmap doesn't just say "open" or "closed" — it has six distinct states:

| State | Meaning |
|---|---|
| **Open** | A connection was established (TCP, UDP, or SCTP). The service is actively accepting connections. |
| **Closed** | The target replied with a TCP `RST` flag. Port is accessible but nothing is listening. Also useful for confirming the host is alive. |
| **Filtered** | No response, or an error code was received. Nmap can't determine open/closed — likely a firewall dropping packets. |
| **Unfiltered** | Occurs only during TCP-ACK scans. Port is reachable, but open/closed status is unknown. |
| **open\|filtered** | No response from the port — a firewall or packet filter may be silently dropping traffic. |
| **closed\|filtered** | Occurs only in IP ID idle scans. Impossible to determine if a firewall is involved. |

---

## Discovering Open TCP Ports

### Scanning the Top 10 Ports

```bash
sudo nmap 10.129.2.28 --top-ports=10
```

| Flag | Description |
|---|---|
| `10.129.2.28` | Target IP address |
| `--top-ports=10` | Scan the 10 most commonly used TCP ports |

This is a quick way to check for the most likely running services without doing a full scan.

---

### Tracing Packets

To see exactly what Nmap is sending and receiving at the packet level:

```bash
sudo nmap 10.129.2.28 -p 21 --packet-trace -n --disable-arp-ping
```

![packet trace output](/assets/img/posts/tracing_packets.png)

| Flag | Description |
|---|---|
| `-p 21` | Scan only port 21 (FTP) |
| `--packet-trace` | Show all packets sent and received |
| `-n` | Disable DNS resolution (faster, no noise) |
| `--disable-arp-ping` | Disable ARP ping |

#### Reading the Packet Trace Output

**Sent packet:**

```
SENT (0.0429s) TCP 10.10.14.2:63090 > 10.129.2.28:21 S ttl=56 id=57322 iplen=44 seq=1699105818 win=1024 mss 1460
```

| Field | Meaning |
|---|---|
| `SENT (0.0429s)` | Nmap sent a packet at this timestamp |
| `TCP` | Protocol in use |
| `10.10.14.2:63090` | Our IP and source port |
| `10.129.2.28:21` | Target IP and port |
| `S` | SYN flag set — initiating connection |
| `ttl=56 id=57322 ...` | Additional TCP header parameters |

**Received packet:**

```
RCVD (0.0573s) TCP 10.129.2.28:21 > 10.10.14.2:63090 RA ttl=64 id=0 iplen=40 seq=0 win=0
```

| Field | Meaning |
|---|---|
| `RCVD (0.0573s)` | Packet received at this timestamp |
| `10.129.2.28:21` | Source: target IP and port |
| `10.10.14.2:63090` | Destination: our IP and port |
| `RA` | RST + ACK flags — port is **closed** |

---

## TCP Scan Types

### Connect Scan (`-sT`) — Full Handshake

The **Connect Scan** completes the full TCP three-way handshake:

```
SYN → SYN-ACK → ACK (connection established)
```

```bash
sudo nmap 10.129.2.28 -p 443 -sT
```

![packet trace output](/assets/img/posts/tcp_sT.png)

- Highly accurate — uses the OS's TCP stack
- **Not stealthy** — creates logs on most systems, easily detected by IDS/IPS

### SYN Scan (`-sS`) — Half-Open / Stealth

The **SYN Scan** never completes the handshake:

```
SYN → SYN-ACK → RST (connection torn down immediately)
```

- Much stealthier — avoids full connection logging
- Faster than a Connect Scan
- Requires root/sudo privileges

---

## Filtered Ports — Firewalls in the Way

A **filtered** port means the firewall is interfering. There are two ways this can manifest:

### Dropped Packets

![packet trace output](/assets/img/posts/drop_packet.png)

Nmap receives no response. It will retry up to `--max-retries` times (default: 10). You'll notice this in the scan time — it takes significantly longer due to timeout waits.

### Rejected Packets

![packet trace output](/assets/img/posts/reject_packet.png)

The target (or firewall) sends back an **ICMP unreachable** message:

```
ICMP type=3, code=3 — Destination Port Unreachable
```

This is faster to detect but leaks the existence of a filtering device.

---

## Discovering Open UDP Ports

UDP is a **stateless protocol** — there's no handshake, no guaranteed response.

```bash
sudo nmap 10.129.2.28 -sU
```
![packet trace output](/assets/img/posts/udp_reject.png)

The challenge with UDP scanning:
- Nmap sends **empty datagrams** to each UDP port
- If the port is **open**, a response only comes if the application is configured to reply
- If the port is **closed**, the OS returns an ICMP error: `type=3, code=3` (port unreachable)
- If there's **no response at all**, the port is marked `open|filtered`

> UDP scans are slow and noisy. Combine with `-Pn` to skip the host discovery ping and focus directly on ports.

```bash
sudo nmap 10.129.2.28 -sU -Pn
```

| Flag | Description |
|---|---|
| `-sU` | UDP scan mode |
| `-Pn` | Skip ICMP echo requests (treat host as up) |

---

## Version Scanning (`-sV`)

Once we know which ports are open, we want to fingerprint the services running on them.

```bash
sudo nmap 10.129.2.28 -Pn -n --disable-arp-ping --packet-trace -p 445 --reason -sV
```

| Flag | Description |
|---|---|
| `10.129.2.28` | Target IP |
| `-Pn` | Disable ICMP echo requests |
| `-n` | Disable DNS resolution |
| `--disable-arp-ping` | Disable ARP ping |
| `--packet-trace` | Show all packets sent and received |
| `-p 445` | Scan only SMB port |
| `--reason` | Show why Nmap assigned a particular state to the port |
| `-sV` | Enable service/version detection |

**What `-sV` gives you:**

![packet trace output](/assets/img/posts/service.png)

- Service name (e.g., `microsoft-ds`, `ftp`, `ssh`)
- Version string (e.g., `OpenSSH 8.2p1`)
- Additional banners or metadata the service exposes

This is often enough to start identifying CVEs or misconfigurations tied to specific software versions.

---

## Quick Reference Cheatsheet

```bash
# Fast scan — top 10 ports
sudo nmap <target> --top-ports=10

# Stealth SYN scan with version detection
sudo nmap <target> -sS -sV -Pn -n

# Full connect scan on a specific port
sudo nmap <target> -sT -p 443

# UDP scan (slow — be patient)
sudo nmap <target> -sU -Pn

# Trace packets for a single port
sudo nmap <target> -p 21 --packet-trace -n --disable-arp-ping

# Version scan with reason and no noise
sudo nmap <target> -sV -Pn -n --disable-arp-ping -p 445 --reason
```

---

## Key Takeaways

- **Port states matter** — don't treat everything that isn't "open" as a dead end. Filtered ports often hide interesting services behind firewalls.
- **Stealth vs. accuracy** — SYN scans are quieter; Connect scans are more reliable. Know when to use each.
- **UDP is often overlooked** — services like DNS (53), SNMP (161), and TFTP (69) run over UDP and are frequently misconfigured.
- **Version detection is powerful** — `-sV` can expose enough information to pivot directly into exploitation research.

---

*HTB Academy — CPTS module notes. All scans performed on dedicated lab machines.*
