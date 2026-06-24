# Wgel Security Assessment Report

## Executive Summary

A penetration test was performed against the Wgel target machine.

The assessment identified multiple security weaknesses that allowed an attacker to:

* Enumerate exposed services.
* Discover valid usernames.
* Obtain an exposed SSH private key.
* Gain remote shell access.
* Abuse sudo permissions.
* Obtain the root flag.

Overall Risk: Critical

---

# Scope

Target:
10.65.137.127

Methodology:

* PTES
* MITRE ATT&CK

Objective:

* Obtain User Flag
* Obtain Root Flag

---

# Finding 1

## Information Disclosure Through HTML Comments

Severity: Low

Evidence:

```html
<!-- Jessie don't forget to update the website -->
```

Impact:
Username disclosure.

MITRE:
T1592

---

# Finding 2

## Sensitive SSH Private Key Exposure

Severity: Critical

Affected Resource:

```
/sitemap/.ssh/id_rsa
```

Impact:

An attacker can authenticate directly through SSH.

MITRE:
T1552

Recommendation:

* Remove private keys from web-accessible directories.
* Restrict directory access.
* Perform periodic secret scanning.

---

# Finding 3

## Unauthorized SSH Access

Severity: High

Authentication succeeded using the exposed private key.

MITRE:
T1021.004

---

# Finding 4

## Sudo Misconfiguration

Severity: Critical

The user jessie could execute wget with elevated privileges.

MITRE:
T1548

Impact:

Sensitive root files could be exfiltrated.

---

# Attack Path

HTTP
↓
HTML Comment
↓
Username Discovery
↓
Directory Enumeration
↓
Exposed SSH Key
↓
SSH Access
↓
Sudo Abuse
↓
Root Flag

---

# MITRE ATT&CK Mapping

| Technique                 | ATT&CK    |
| ------------------------- | --------- |
| Service Discovery         | T1046     |
| Active Scanning           | T1595     |
| Gather Victim Information | T1592     |
| Unsecured Credentials     | T1552     |
| SSH Access                | T1021.004 |
| Abuse Elevation Control   | T1548     |

---

# Conclusion

The compromise of the target was achieved through a chain of relatively simple security weaknesses.

No software vulnerabilities were required.

The attack relied entirely on:

* Information disclosure.
* Credential exposure.
* Improper privilege configuration.

These weaknesses combined to allow complete compromise of the system.
