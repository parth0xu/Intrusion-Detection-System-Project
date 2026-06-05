# snort.conf — Key Changes & Rationale

## Changes Made

### 1. HOME_NET (Line ~45 in snort.conf)

```
# Before (default)
ipvar HOME_NET any

# After
ipvar HOME_NET 192.168.16.0/24
```

**Why**: Scoping HOME_NET to the actual lab subnet ensures rules with `-> $HOME_NET` only
fire for traffic destined to our protected machines. Using `any` would generate alerts
for all traffic including external, creating noise.

---

### 2. EXTERNAL_NET

```
ipvar EXTERNAL_NET any
```

Left as `any` — standard for most lab setups. In production, this would be set to
`!$HOME_NET` to explicitly define everything outside the protected subnet as external.

---

### 3. Rule Path Verification

```
var RULE_PATH /etc/snort/rules
```

Confirmed `include $RULE_PATH/local.rules` is uncommented in Step 7 section of the config.
Community and most default rule categories remain disabled (prefixed with `#include`) to
reduce noise during lab testing. Only the following were active:

- `chat.rules`
- `ddos.rules`
- `dns.rules`
- `dos.rules`
- `experimental.rules`
- `exploit.rules`
- `local.rules` ← our custom rules

---

## Snort Debian Config Override

On Ubuntu/Debian, `/etc/snort/snort.debian.conf` overrides HOME_NET at runtime when
Snort is started via the init.d daemon. To ensure our snort.conf value is respected
when running Snort manually (not via daemon), we run directly:

```bash
sudo snort -A console -i enp0s3 -c /etc/snort/snort.conf
```

---

## Self-Test Command

```bash
sudo snort -T -i enp0s3 -c /etc/snort/snort.conf
```

Output confirming config is valid:
```
Snort successfully validated the configuration!
Snort exiting
```
