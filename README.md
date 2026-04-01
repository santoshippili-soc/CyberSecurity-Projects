# 🔥 pfSense Firewall Home Lab — DoS Defense Demo

A hands-on cybersecurity home lab simulating a **Denial-of-Service (DoS) attack** and its mitigation using **pfSense**, **Kali Linux**, and **Ubuntu**, all running in **VirtualBox**.

---

## 📌 Overview

This lab demonstrates:
- Setting up a two-zone firewall environment with pfSense
- Simulating a DoS/flood attack from an external attacker (Kali Linux)
- Detecting the attack using Wireshark on the victim machine
- Blocking the attack by creating a firewall rule in pfSense
- Verifying the block through pfSense firewall logs

---

## 📸 Lab Screenshots

| Step | Screenshot | Description |
|---|---|---|
| 1 | ![pfSense Console](screenshots/08-pfsense-console-pfctl.png) | pfSense console — interface assignment (WAN: 192.168.1.6, LAN: 192.168.10.1) and `pfctl -d` to disable firewall for initial GUI access |
| 2 | ![pfSense Login](screenshots/01-pfsense-login.png) | pfSense WebGUI login page accessed from Kali over WAN (https://192.168.1.14) |
| 3 | ![Ubuntu IP + Internet](screenshots/02-ubuntu-ip-internet.png) | Ubuntu victim VM — IP `192.168.10.100` assigned via pfSense DHCP, internet connectivity verified with `ping 8.8.8.8` and `ping google.com` |
| 4 | ![pfSense WAN Rules](screenshots/03-pfsense-wan-rules.png) | pfSense WAN firewall rules — block rule (top, red ✗) and allow rule for Kali→Ubuntu in place |
| 5 | ![Kali hping3 Flood](screenshots/07-kali-hping3-flood.png) | Kali Linux launching SYN flood using `hping3 --flood -S -p 80 192.168.10.100` |
| 6 | ![Wireshark SYN Flood](screenshots/04-wireshark-syn-flood.png) | Wireshark on Ubuntu capturing 18,281 packets — SYN flood from `192.168.1.5` → `192.168.10.100` visible |
| 7 | ![Block Rule Applied](screenshots/06-pfsense-block-rule-applied.png) | pfSense WAN rules after block rule placed at top — 0/29.89 MiB blocked traffic counter |
| 8 | ![Wireshark After Block](screenshots/05-wireshark-after-block.png) | Wireshark after block rule applied — only 14 normal packets visible, flood traffic completely stopped |

---

## 🗺️ Network Topology

```
Internet / Home LAN
        │
  ┌─────┴──────┐
  │  Kali Linux │  (Attacker)
  │ 192.168.1.5 │  [Bridged Adapter]
  └─────┬──────┘
        │
  ┌─────┴──────────────┐
  │   pfSense Firewall  │
  │  WAN: 192.168.1.6   │  [Bridged Adapter → vtnet0]
  │  LAN: 192.168.10.1  │  [Internal Network → vtnet1]
  └─────┬──────────────┘
        │  Internal Network: LabNet (192.168.10.0/24)
  ┌─────┴──────────┐
  │ Ubuntu Desktop  │  (Victim)
  │ 192.168.10.100  │  [Internal Network Adapter]
  └────────────────┘
```

| Device | Adapter | Interface | IP Address | Role |
|---|---|---|---|---|
| Kali Linux | Bridged | eth0 | 192.168.1.5 | External Attacker |
| pfSense WAN | Bridged | vtnet0 | 192.168.1.6 | Edge Firewall (WAN) |
| pfSense LAN | Internal (LabNet) | vtnet1 | 192.168.10.1 | Default GW + DHCP |
| Ubuntu Desktop | Internal (LabNet) | eth0 | 192.168.10.100 | Victim Host |

---

## 🛠️ Prerequisites

- VirtualBox ≥ 7.x
- ISO images:
  - [pfSense CE](https://www.pfsense.org/download/) (pfSense-CE-2.7.x-amd64.iso)
  - [Kali Linux](https://www.kali.org/get-kali/) (kali-linux-current-amd64.iso)
  - [Ubuntu Desktop](https://ubuntu.com/download/desktop) (ubuntu-22.04-desktop-amd64.iso)
- Host minimum: **8 GB RAM**, **40 GB free disk**
- Admin rights on the host machine

---

## 🖥️ VM Configuration

### pfSense VM
| Setting | Value |
|---|---|
| OS Type | FreeBSD 64-bit |
| CPUs / RAM | 2 vCPU / 2 GB |
| Disk | 20 GB VDI (dynamic) |
| Adapter 1 | Bridged → Physical NIC (WAN) |
| Adapter 2 | Internal Network → LabNet (LAN) |

### Kali Linux VM (Attacker)
| Setting | Value |
|---|---|
| OS Type | Debian 64-bit |
| CPUs / RAM | 2 vCPU / 2 GB |
| Disk | 15 GB VDI (dynamic) |
| Adapter 1 | Bridged (WAN side) |

### Ubuntu VM (Victim)
| Setting | Value |
|---|---|
| OS Type | Ubuntu 64-bit |
| CPUs / RAM | 2 vCPU / 2 GB |
| Disk | 15 GB VDI (dynamic) |
| Adapter 1 | Internal Network → LabNet |

---

## ⚙️ Lab Setup Steps

### 1. pfSense Installation & Interface Assignment

- Boot pfSense VM from ISO and accept defaults during install
- From the console menu, assign interfaces:
  - `vtnet0` → WAN (Bridged)
  - `vtnet1` → LAN (LabNet)
- Leave WAN on DHCP — pfSense will receive **192.168.1.6** from the home router

### 2. Access pfSense WebGUI from the WAN

Access the pfSense shell and temporarily disable the firewall to allow initial GUI access:
```bash
pfctl -d
```
- Browse to `https://192.168.1.6` from Kali or host
- Default credentials: `admin` / `pfsense`
- Add a WAN firewall rule to allow GUI access:
  - **Firewall → Rules → WAN → Add**
  - Action: Pass | Protocol: TCP | Source: Home Network | Destination: pfSense WAN | Port: 443

### 3. Configure LAN DHCP

- Navigate to **Services → DHCP Server → LAN**
- Enable DHCP on LAN interface
- Range: `192.168.10.100` – `192.168.10.199`
- DNS Server: `192.168.1.1` (home router)
- Save and Apply

### 4. Allow Kali Access to Internal LAN (for Demo)

- **Firewall → Rules → WAN → Add**
- Action: Pass | Protocol: Any | Source: `192.168.1.5` (Kali) | Destination: `192.168.10.100` (Ubuntu)
- Description: `Allow Kali access to Ubuntu`

### 5. Kali Linux — Add Static Route

Check IP:
```bash
ifconfig
```

Add a route to reach the internal LAN via pfSense WAN:
```bash
sudo ip route add 192.168.10.0/24 via 192.168.1.6
```

### 6. Ubuntu (Victim) — Verify Connectivity

Ubuntu should automatically receive `192.168.10.100` via pfSense DHCP.

```bash
ifconfig
ping -c 3 google.com
```

Test from Kali → Ubuntu:
```bash
ping 192.168.10.100
```
This should succeed before the attack.

---

## 💥 Launching the DoS Attack (hping3)

### On Ubuntu — Start Wireshark Capture

```bash
sudo apt update && sudo apt install -y wireshark
sudo wireshark &   # Capture on eth0
```

### On Kali — Launch the Flood

ICMP Flood:
```bash
sudo hping3 -1 --flood 192.168.10.100
```

SYN Flood (alternative):
```bash
sudo hping3 --flood -S -p 80 192.168.10.100
```

Observe the **packet spike** in Wireshark on Ubuntu — thousands of packets per second flooding in.

---

## 🛡️ Blocking the Attack in pfSense

1. Navigate to **Firewall → Rules → WAN → Add (TOP)**
2. Configure the block rule:
   - **Action:** Block
   - **Source:** `192.168.1.5` (Kali IP)
   - **Destination:** `192.168.10.100` (Ubuntu IP)
   - **Log:** ✅ Enabled
   - **Description:** `Block Kali DoS`
3. Click **Apply Changes**

✅ Traffic halts — Wireshark on Ubuntu should stop showing the flood traffic immediately.

### Verify in pfSense Logs

- Navigate to **Status → System Logs → Firewall**
- Filter by the `Block Kali DoS` rule
- You will see blocked entries confirming the flood packets are being dropped

---

## 🧰 Tools Used

| Tool | Purpose |
|---|---|
| pfSense CE | Firewall / Router / DHCP Server |
| Kali Linux | Attacker machine |
| hping3 | DoS flood packet generator |
| Ubuntu Desktop | Victim machine |
| Wireshark | Packet capture and traffic visualization |
| VirtualBox | Virtualization platform |

---

## 🔐 Key Concepts Demonstrated

- **Firewall rule ordering** — Block rules placed at the TOP take priority over allow rules below
- **WAN vs LAN rule placement** — Inbound attack traffic is filtered at the WAN interface
- **Firewall logging** — Enabling logs on rules provides evidence of blocked malicious traffic
- **Static routing** — Manual route injection on Kali to reach the isolated LAN segment
- **DHCP scoping** — pfSense acting as DHCP server for the internal LabNet subnet
- **Network segmentation** — WAN and LAN zones isolate the victim from the attacker

---

## 🔒 Security Best Practices

- Keep the homelab **isolated from production/home networks**
- Regularly update Ubuntu, Kali, and pfSense to patch known vulnerabilities
- Always **re-enable the pfSense firewall** (`pfctl -e`) after initial setup
- Use **strong credentials** — change the default `admin/pfsense` password immediately
- Enable **logging** on critical firewall rules for audit trails

---

## 📁 Repository Structure

```
pfSense-DoS-Lab/
├── README.md               ← This file
├── network-diagram.md      ← Detailed topology and IP plan
├── firewall-rules.md       ← pfSense rule configurations
└── attack-and-defense.md   ← Attack commands and mitigation steps
```

---

## 👤 Author

**Santosh Ippili**  
B.Tech CSE (Cybersecurity) — VIT-AP University, 2025  
EC-Council Certified SOC Analyst (CSA) | AWS Certified Cloud Practitioner  

---

## 📜 License

This project is for **educational purposes only**. Do not use these techniques on systems you do not own or have explicit permission to test.
