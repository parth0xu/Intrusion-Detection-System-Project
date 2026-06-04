# Intrusion Detection System — Snort + ELK Stack

> **Network security monitoring lab** built to detect, alert, and visualize real-time intrusion attempts using Snort IDS integrated with the ELK Stack. Simulated attacks using Nmap and Metasploit to validate custom rule effectiveness.

---

## What This Project Does

This project sets up a functional IDS environment on a Linux host that:

- Monitors live network traffic and fires alerts on suspicious patterns
- Uses **custom Snort rules** to detect specific attack signatures (port scans, SYN floods, ICMP sweeps)
- Ships alert logs to **Elasticsearch via Logstash** and visualizes them on a **Kibana dashboard**
- Validates detection by simulating real attacks from an attacker machine (Kali Linux)

---

## Architecture

```
Attacker Machine (Kali)
        │
        │  ← Nmap SYN scan / Metasploit exploit attempt
        ▼
[ Network Interface (eth0) ]
        │
        ▼
   Snort IDS (Ubuntu)
   ├── Custom rules → /etc/snort/rules/local.rules
   ├── Alerts → /var/log/snort/alert
        │
        ▼
   Logstash (log shipper)
        │
        ▼
   Elasticsearch → Kibana Dashboard
```

---

## Tech Stack

| Tool | Purpose |
|------|---------|
| Snort 2.9.x | Core IDS engine — packet inspection & alerting |
| Wireshark | Packet capture & manual traffic analysis |
| Nmap | Attack simulation — port scanning, OS fingerprinting |
| Metasploit | Exploit simulation for detection validation |
| ELK Stack (Elasticsearch + Logstash + Kibana) | Log aggregation, storage, and dashboarding |
| Ubuntu 22.04 | IDS host OS |
| Kali Linux | Attacker machine (VM) |

---

## Setup & Installation

### Prerequisites

- Two machines (or VMs) on the same network: one Ubuntu (IDS host), one Kali (attacker)
- At least 4GB RAM on the IDS host for ELK Stack

### 1. Install Snort

```bash
sudo apt update && sudo apt install -y snort
```

Verify installation:

```bash
snort --version
```

### 2. Configure Snort

Set your network range in `/etc/snort/snort.conf`:

```bash
sudo nano /etc/snort/snort.conf
# Set HOME_NET to your local subnet, e.g.:
# var HOME_NET 192.168.1.0/24
```

### 3. Write Custom Detection Rules

Add rules to `/etc/snort/rules/local.rules`:

```
# Detect Nmap SYN scan
alert tcp any any -> $HOME_NET any (msg:"Nmap SYN Scan Detected"; flags:S; threshold:type threshold, track by_src, count 20, seconds 1; sid:1000001; rev:1;)

# Detect ICMP ping sweep
alert icmp any any -> $HOME_NET any (msg:"ICMP Ping Sweep"; itype:8; threshold:type threshold, track by_src, count 5, seconds 2; sid:1000002; rev:1;)

# Detect SSH brute force attempt
alert tcp any any -> $HOME_NET 22 (msg:"SSH Brute Force Attempt"; flow:to_server; threshold:type threshold, track by_src, count 5, seconds 60; sid:1000003; rev:1;)
```

### 4. Run Snort in IDS Mode

```bash
sudo snort -A console -q -c /etc/snort/snort.conf -i eth0
```

### 5. Simulate an Attack (from Kali)

```bash
# SYN scan — should trigger rule 1000001
nmap -sS 192.168.1.x

# Ping sweep — should trigger rule 1000002
nmap -sn 192.168.1.0/24
```

### 6. View Raw Alerts

```bash
cat /var/log/snort/alert
```

---

## ELK Stack Integration

### Ship Snort logs to Elasticsearch via Logstash

Create `/etc/logstash/conf.d/snort.conf`:

```
input {
  file {
    path => "/var/log/snort/alert"
    start_position => "beginning"
  }
}
output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "snort-alerts-%{+YYYY.MM.dd}"
  }
}
```

Start Logstash:

```bash
sudo systemctl start logstash
```

Then open Kibana at `http://localhost:5601` and create an index pattern for `snort-alerts-*`.

---

## What I Learned

- How Snort's rule engine processes packets and why rule ordering matters for performance
- Writing threshold-based rules to reduce false positives (critical in real SOC environments)
- How to correlate IDS alerts with raw pcap data in Wireshark to confirm true positives
- Why ELK integration matters — raw alert logs are unworkable at scale; dashboards surface patterns

---

## Full Project Report

For detailed methodology, screenshots, and test results: [`IDS Snort project.pdf`](./IDS%20Snort%20project.pdf)

---

## Skills Demonstrated

`Snort` `IDS/IPS` `Custom Rule Writing` `ELK Stack` `Log Analysis` `Network Traffic Analysis` `Wireshark` `Nmap` `Incident Response` `Linux`
