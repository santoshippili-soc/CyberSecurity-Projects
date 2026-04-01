# 🛡️ pfSense SYN Flood Attack – Detection & Mitigation Lab

A hands-on home lab demonstrating a **TCP SYN Flood DoS attack** launched from a Kali Linux machine, captured with Wireshark on an Ubuntu target, and mitigated using pfSense firewall rules — all running inside VirtualBox.

---

## 🧪 Lab Environment

| Component | Role | IP Address |
|-----------|------|------------|
| pfSense 2.8.1 (VirtualBox VM) | Firewall / Router | WAN: `192.168.1.6/24` · LAN: `192.168.10.1/24` |
| Kali Linux | Attacker | `192.168.1.5` |
| Ubuntu (VirtualBox VM) | Target / Victim | `192.168.10.100` |

---

## 🎯 Objective

- Simulate a real TCP SYN Flood denial-of-service attack in a safe, isolated environment
- Capture and analyse the malicious traffic using Wireshark
- Block the attack at the firewall level using pfSense WAN rules
- Validate that the mitigation is effective by comparing pre- and post-block traffic

---

## 📋 Lab Steps

### 1. pfSense Setup & Login
- Access the pfSense web GUI at `https://<WAN_IP>/index.php`
- Log in with admin credentials
- Verify interface assignments:
  - **WAN** → `em0` → `192.168.1.6/24` (DHCP)
  - **LAN** → `em1` → `192.168.10.1/24`

---

### 2. Verify Target Connectivity (Ubuntu)
Before launching the attack, confirm the Ubuntu target is online and properly routed.

```bash
ifconfig          # Verify IP assigned: 192.168.10.100
ping 8.8.8.8      # Test external reachability
ping google.com   # Test DNS resolution
```

**Result:** 0% packet loss with ~45 ms RTT — normal connectivity confirmed.

---

### 3. Launch SYN Flood from Kali (hping3)
From the Kali attacker machine, send a high-volume flood of TCP SYN packets to port 80 of the target.

```bash
sudo hping3 --flood -S -p 80 192.168.10.100
```

**Flags explained:**

| Flag | Description |
|------|-------------|
| `--flood` | Send packets as fast as possible, no reply display |
| `-S` | Set the TCP SYN flag |
| `-p 80` | Target port 80 (HTTP) |

**Output:**
```
HPING 192.168.10.100 (eth0 192.168.10.100): S set, 40 headers + 0 data bytes
hping in flood mode, no replies will be shown
```

---

### 4. Capture & Analyse Traffic (Wireshark on Ubuntu)
Wireshark was running on interface `enp0s3` on the Ubuntu machine during the attack.

**Observations during the flood:**
- Massive stream of `[SYN]` packets incoming from `192.168.1.5 → 192.168.10.100:80`
- Target responded to each SYN with `[RST, ACK]` — classic half-open connection exhaustion behaviour
- Over **18,000 packets** captured in a short window
- Each SYN packet: 54 bytes, Seq=0, Win=512

This pattern is the signature of a **TCP SYN Flood** — the attacker never completes the three-way handshake, keeping the target busy processing unanswered connection requests.

---

### 5. Apply WAN Block Rule in pfSense
Navigate to **Firewall → Rules → WAN** and add a block rule for the attacker's source IP.

**Rule configuration:**

| Field | Value |
|-------|-------|
| Action | ❌ Block |
| Protocol | IPv4 TCP |
| Source | `192.168.1.5` (attacker) |
| Destination | `192.168.10.100` (target) |
| Port | `*` (any) |
| Queue | none |

> ⚠️ The block rule must be positioned **above** any existing allow rules in the list so it is evaluated first.

**Final WAN rule order (top to bottom):**

| # | Action | Protocol | Source | Destination |
|---|--------|----------|--------|-------------|
| 1 | ❌ Block | IPv4 TCP | 192.168.1.5 | 192.168.10.100 |
| 2 | ✅ Allow | IPv4 ICMP any | 192.168.1.5 | 192.168.1.6 |
| 3 | ✅ Allow | IPv4 TCP | 192.168.1.0/24 | 192.168.1.6 |
| 4 | ✅ Allow | IPv4 TCP | 192.168.1.5 | 192.168.10.100 |

---

### 6. Reload Packet Filter via pfSense Console
Access the pfSense shell through the console menu (Option 8 → Shell) and verify or reload the packet filter.

```bash
pfctl -d    # Disable pf (used during testing to observe unfiltered traffic)
pfctl -e    # Re-enable pf to apply updated rules
```

---

### 7. Validate Mitigation (Wireshark Post-Block)
After the firewall rule was applied, Wireshark on Ubuntu showed a dramatic change in traffic.

**Post-block observations:**
- Zero SYN flood packets from `192.168.1.5`
- Only legitimate traffic visible: DNS queries, proper TCP three-way handshakes, HTTP GET requests
- Traffic dropped from **18,000+ packets → 14 packets**
- Ubuntu successfully reached `connectivity-check.ubuntu.com` (HTTP 204 No Content response)

---

## ✅ Results Summary

| Phase | Traffic | Observation |
|-------|---------|-------------|
| Baseline | Normal | ~45 ms RTT, 0% packet loss, clean connectivity |
| During SYN Flood | 18,000+ packets | Flood of SYN → RST/ACK responses, port 80 overwhelmed |
| After Firewall Rule | 14 packets | Attack traffic blocked; only legitimate sessions visible |

---

## 🔧 Tools & Technologies

| Tool | Purpose |
|------|---------|
| **pfSense 2.8.1** | Open-source firewall and router |
| **hping3** | TCP/IP packet crafting and flood tool (Kali Linux) |
| **Wireshark** | Real-time network packet capture and analysis |
| **VirtualBox** | Virtualisation platform for the isolated lab |
| **Ubuntu** | Target machine / Wireshark host |
| **Kali Linux** | Attacker machine |

---

## 💡 Key Concepts

**What is a TCP SYN Flood?**

A SYN Flood exploits the TCP three-way handshake. The attacker sends a large number of SYN packets but never completes the handshake (never sends ACK). This forces the target to keep half-open connections in memory until they time out, eventually exhausting resources.

```
Normal Handshake:      SYN Flood:
Client  →  SYN         Attacker → SYN (×thousands)
Server  →  SYN-ACK     Server   → SYN-ACK (never acknowledged)
Client  →  ACK         [connection table fills up]
[Connected]            [Legitimate users denied]
```

**Why does pfSense block it effectively?**

pfSense evaluates WAN rules before traffic reaches the internal LAN. A block rule on the attacker's source IP drops all matching packets at the perimeter, so the victim machine never sees the flood.

---

## 📌 Key Takeaways

- TCP SYN floods are simple to launch but devastating without proper firewall protection
- Stateful firewalls like pfSense can neutralise flood attacks at the network boundary
- Wireshark is an invaluable tool for both confirming an attack is happening and verifying a mitigation worked
- Rule order in pfSense matters — block rules must precede allow rules for the same traffic

---

> **⚠️ Disclaimer:** This lab was conducted in a fully isolated virtual environment for educational and research purposes only. Never perform these techniques on networks or systems you do not own or have explicit written permission to test.
