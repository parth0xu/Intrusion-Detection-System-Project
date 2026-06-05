# Snort IDS Lab — Network Intrusion Detection on Ubuntu

A hands-on implementation of Snort 2.9 as a Network Intrusion Detection System (NIDS) on Ubuntu Linux using VirtualBox. This lab demonstrates real-time traffic monitoring, custom rule creation, and alert generation against a Metasploitable 2 target machine.

---

## Lab Environment

| Component | Details |
|---|---|
| Host OS | Windows (VirtualBox host) |
| IDS Machine | Ubuntu Linux (VirtualBox VM) |
| Target Machine | Metasploitable 2 (VirtualBox VM) |
| Snort Version | 2.9.20 |
| Network Adapter | Bridged Adapter — `enp0s3` |
| Promiscuous Mode | Allow All |
| HOME_NET | `192.168.16.0/24` |

Both VMs are on the same bridged network, allowing Snort running on Ubuntu to monitor traffic between the attacker (Metasploitable) and the IDS machine.

---

## Setup & Configuration

### 1. Install Snort

```bash
sudo apt-get install snort -y
```

During installation, two prompts appear:
- **Interface**: Set to `enp0s3` (the active network interface — verify with `ifconfig`)
- **HOME_NET**: Set to your subnet in CIDR notation (e.g., `192.168.2.0/24`)

### 2. Enable Promiscuous Mode (VirtualBox)

In VirtualBox VM settings → Network → Advanced:
- Set **Promiscuous Mode** to `Allow All`

This allows the NIC to capture all packets on the network segment, not just those addressed to the VM — essential for IDS functionality.

### 3. Verify Snort Config Directory

```bash
ls -al /etc/snort
```

Key files:
- `snort.conf` — main configuration file
- `rules/local.rules` — where custom rules go
- `snort.debian.conf` — Debian-specific overrides (HOME_NET defined here on Debian/Ubuntu)

### 4. Edit snort.conf — Set HOME_NET

```bash
sudo vim /etc/snort/snort.conf
```

Find and update:

```
ipvar HOME_NET 192.168.16.0/24
ipvar EXTERNAL_NET any
```

This tells Snort which IP range to treat as the protected internal network.

### 5. Verify local.rules is Included

In `snort.conf`, confirm this line exists and is **not** commented out:

```
include $RULE_PATH/local.rules
```

### 6. Test Configuration

```bash
sudo snort -T -i enp0s3 -c /etc/snort/snort.conf
```

Expected output: `Snort successfully validated the configuration!`

---

## Custom Rules

Rules are written in `/etc/snort/rules/local.rules`.

```bash
sudo vim /etc/snort/rules/local.rules
```

### Rule Syntax

```
action protocol src_ip src_port -> dst_ip dst_port (options)
```

| Field | Description |
|---|---|
| `action` | `alert`, `log`, `drop`, `reject` |
| `protocol` | `tcp`, `udp`, `icmp`, `ip` |
| `msg` | Human-readable alert message |
| `sid` | Unique rule ID (local rules use 10000+) |
| `rev` | Rule revision number |

---

### Rules Written

**Rule 1 — ICMP Ping Detection**
```
alert icmp any any -> $HOME_NET any (msg:"ICMP Ping Detected"; sid:10001; rev:1;)
```
Detects any ICMP echo request directed at the home network. In real scenarios, ping sweeps are used by attackers for host discovery / network reconnaissance.

**Rule 2 — SSH Authentication Attempt**
```
alert tcp any any -> $HOME_NET 22 (msg:"SSH Authentication Attempt"; sid:10002; rev:1;)
```
Triggers on any TCP traffic to port 22. Useful for detecting SSH brute force attempts or unauthorized access attempts against SSH services.

**Rule 3 — FTP Authentication Attempt** *(created via Snorpy 2.0)*
```
alert tcp any any -> $HOME_NET 21 (msg:"FTP Authentication Attempt"; sid:100003; rev:1;)
```
Detects connection attempts to FTP port 21. FTP transmits credentials in plaintext — any auth attempt to this port warrants logging.

---

## Running Snort in NIDS Mode

```bash
sudo snort -A console -i enp0s3 -c /etc/snort/snort.conf
```

| Flag | Purpose |
|---|---|
| `-A console` | Print alerts to stdout in real time |
| `-i enp0s3` | Listen on this interface |
| `-c` | Use this config file |

---

## Alert Output — Proof of Detection

From Metasploitable 2, a ping sweep was run targeting the Ubuntu IDS machine:

```
msfadmin@metasploitable:~$ ping -c5 192.168.16.24
```

Snort immediately triggered the ICMP rule and logged bidirectional traffic:

```
02/10-17:15:13.623259 [**] [1:10001:1] ICMP Ping Detected [**] [Priority: 0] {ICMP} 192.168.16.120 -> 192.168.16.24
02/10-17:15:13.623321 [**] [1:10001:1] ICMP Ping Detected [**] [Priority: 0] {ICMP} 192.168.16.24 -> 192.168.16.120
```

Each alert line contains:
- **Timestamp** — exact time of packet
- **Rule SID** — `[1:10001:1]` maps to our custom rule
- **Message** — `ICMP Ping Detected`
- **Protocol** — `{ICMP}`
- **Source → Destination** — IP pair showing both directions

---

## Snorpy 2.0 — Web-Based Rule Builder

[Snorpy](http://snorpy.cyb3rs3c.net/) provides a GUI for constructing Snort rules without memorizing syntax. Used here to generate the FTP detection rule. Useful for quick rule prototyping before manually refining in `local.rules`.

---

## Key Takeaways

- **Promiscuous mode** is non-negotiable for a functional IDS — without it, the NIC drops packets not addressed to it and Snort sees nothing
- **HOME_NET scoping** matters: rules with `-> $HOME_NET` only fire for inbound traffic; bidirectional monitoring requires `any` on both sides or separate rules
- **SID namespacing**: SIDs below 1000000 are Snort-reserved; local custom rules should start at 1000000+ (or use 10000+ range for lab environments)
- **Alert noise**: ICMP rules in production would generate enormous false positives — in real deployments, rules are tuned with thresholds and suppression lists

---

## What's Next

- [ ] Add Snort 3 setup and compare rule syntax differences
- [ ] Write rules for Nmap scan detection (SYN scan, NULL scan, XMAS scan)
- [ ] Integrate with a SIEM (e.g., ELK stack) for log visualization
- [ ] Test IPS mode with `inline` and `drop` actions
- [ ] Build detection rules for common Metasploit payloads

---

## Screenshots

| Step | Screenshot |
|---|---|
| Snort Installation | `01_install.png` |
| Interface Config | `02_interface_config.png` |
| HOME_NET Config | `03_homenet_config.png` |
| Promiscuous Mode | `04_promiscuous_mode.png` |
| /etc/snort Directory | `05_snort_dir.png` |
| snort.conf HOME_NET | `06_snortconf_homenet.png` |
| local.rules include | `07_localrules_include.png` |
| Enabled rule categories | `08_rule_categories.png` |
| Config self-test | `09_selftest.png` |
| Opening local.rules | `10_open_localrules.png` |
| Custom rules written | `11_custom_rules.png` |
| Snorpy rule builder | `12_snorpy.png` |
| Snort running + ping from Metasploitable | `13_snort_running.png` |
| ICMP alerts firing | `14_icmp_alerts.png` |

---

## References

- [Snort Official Documentation](https://www.snort.org/documents)
- [Snort Rule Writing Guide](https://docs.snort.org/rules/)
- [Snorpy 2.0 Web Rule Creator](http://snorpy.cyb3rs3c.net/)
- [Metasploitable 2 Setup](https://docs.rapid7.com/metasploit/metasploitable-2/)
