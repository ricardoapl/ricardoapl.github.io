---
layout: post
title: Notes on Hack The Box - Nibbles
---

In this post, I share some notes from my attempt at solving the [Nibbles](https://app.hackthebox.com/machines/Nibbles) machine from [Hack The Box](https://www.hackthebox.com). The following sections describe my solution to this machine in the context of the penetration testing stages presented in the [Penetration Testing Process](https://academy.hackthebox.com/module/90) module from [Hack The Box Academy](https://academy.hackthebox.com), namely:

- Information gathering
- Vulnerability assessment
- Exploitation
- Post-exploitation

I've omitted the pre-engagement, lateral movement, proof-of-concept, and post-engagement stages because they don't seem to be relevant nor applicable.

## Information gathering

In this stage, we gather as much information as possible about our target. Information gathering can be divided into the following categories:

- Open-source intelligence (or OSINT for short)
- Infrastructure enumeration
- Service enumeration
- Host enumeration

There's not much to do when it comes to open-source intelligence or infrastructure enumeration, so I will omit those categories. In the following subsections, I will describe service and host enumeration, and how I've identified:

- Open ports and services
- Service versions
- Operating system

### TCP SYN scan

Given the IP address of the target, we start service and host enumeration by performing a [TCP SYN (Stealth) Scan (-sS)](https://nmap.org/book/synscan.html). This type of scan requires elevated privileges, because Nmap will read and write raw packets instead of using the [`connect()`](https://man7.org/linux/man-pages/man2/connect.2.html) system call from the operating system. In this type of scan, Nmap will send a TCP packet to the target with the SYN flag set in an attempt to begin the [TCP 3-way handshake](https://wiki.wireshark.org/TCP_3_way_handshaking). Nmap will then interpret the responses as follows:

- If the target replies with the SYN and ACK flags set, then the port is in the open state
- If the target replies with the RST flag set, then the port is in the closed state
- If the target doesn't reply, then the port is in the filtered state

Nmap will not complete the TCP 3-way handshake, and instead will send a RST packet to reset the connection.

Enter the following command into the terminal to perform the TCP SYN scan:

```
$ nmap -sV --open -oA nibbles_initial_scan 10.10.10.75
```

The preceding command consists of the following elements:

- `nmap` is the command name.
- `-sV` is the option to enable service and application version detection.
- `--open` is the option to only show the open ports.
- `-oA` is the flag to store scan results in normal, XML, and grepable formats at once.
- `nibbles_initial_scan` specifies the filename that results should be stored in.
- `10.10.10.75` specifies the target host.

The output is similar to the following:

```
Nmap scan report for 10.10.10.75
Host is up (0.056s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
```

We can use the `-O` option from Nmap to perform [OS Detection](https://nmap.org/book/man-os-detection.html). However, the previous command output already hints that the target is running Ubuntu Xenial, because that's the version with [openssh 1:7.2p2-4ubuntu2.2 source package in Ubuntu](https://launchpad.net/ubuntu/+source/openssh/1:7.2p2-4ubuntu2.2).

### TCP SYN scan whole port range

By default, Nmap scans the most common 1000 ports for each protocol using a [Well Known Port List: nmap-services](https://nmap.org/book/nmap-services.html). For a more thorough assessment, Nmap can scan ports from 1 through 65535 using the `-p-` option. It can take a while for Nmap to scan the whole port range, so we can leave this scan running in the background while we explore the web service and application that is running on TCP port 80.

Enter the following command into the terminal to perform the TCP SYN scan whole port range:

```
$ nmap -p- --open -oA nibbles_full_tcp_scan 10.10.10.75
```

The preceding command consists of the following elements:

- `nmap` is the command name.
- `-p-` is the option to scan ports from 1 through 65535.
- `--open` is the option to only show the open ports.
- `-oA` is the flag to store scan results in normal, XML, and grepable formats at once.
- `nibbles_full_tcp_scan` specifies the filename that results should be stored in.
- `10.10.10.75` specifies the target host.

The output is similar to the following:

```
Nmap scan report for 10.10.10.75
Host is up (0.046s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

The aforementioned command output displays nothing new compared to the output of the previous TCP SYN scan.

### Script scan with default set

Knowing what ports the target is listening on, we can leverage the [Nmap Scripting Engine](https://nmap.org/book/nse.html) to perform a script scan using the default [Script Categories](https://nmap.org/book/nse-usage.html#nse-categories). The Nmap Scripting Engine is designed to automate various tasks, and it allows Nmap to gather even more information about the target such as service vulnerabilities.

Enter the following command into the terminal to perform the script scan with default set:

```
$ nmap -sC -p 22,80 -oA nibbles_script_scan 10.10.10.75
```

The preceding command consists of the following elements:

- `nmap` is the command name.
- `-sC` is the option to perform a script scan using the default set of scripts.
- `-p` is the option to specify which ports to scan.
- `-oA` is the flag to store scan results in normal, XML, and grepable formats at once.
- `nibbles_script_scan` specifies the filename that results should be stored in.
- `10.10.10.75` specifies the target host.

The output is similar to the following:

```
Nmap scan report for 10.10.10.75
Host is up (0.046s latency).

PORT   STATE SERVICE
22/tcp open  ssh
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
80/tcp open  http
|_http-title: Site doesn't have a title (text/html).
```

<!-- TODO (ricardoapl): Add a comment/paragraph about scan output -->

### Script scan with http-enum

<!-- TODO (ricardoapl): Explain with https://nmap.org/nsedoc/scripts/http-enum.html -->

Enter the following command into the terminal to perform the script scan with http-enum:

```
$ nmap -sV --script=http-enum -oA nibbles_nmap_http_enum 10.10.10.75
```

The preceding command consists of the following elements:

- `nmap` is the command name.
- `-sV` is the option to enable service and application version detection.
- `--script` is the option to choose which scripts to execute.
- `-oA` is the flag to store scan results in normal, XML, and grepable formats at once.
- `nibbles_nmap_http_enum` specifies the filename that results should be stored in.
- `10.10.10.75` specifies the target host.

The output is similar to the following:

```
Nmap scan report for 10.10.10.75
Host is up (0.047s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
```

### Directory and file brute-force

<!-- TODO (ricardoapl): Add gobuster scan section -->

...

## Vulnerability assessment

<!-- XXX (ricardoapl): Mention searchsploit into NIST -->
<!-- TODO (ricardoapl): Add reference/quote to https://nvd.nist.gov/vuln/detail/CVE-2019-11231 -->

In this stage, ...

## Exploitation

<!-- TODO (ricardoapl): Demonstrate manual exploitation -->
<!-- TODO (ricardoapl): Demonstrate alternative exploitation with metasploit -->

In this stage, ...

## Post-exploitation

In this stage, ...

## Conclusion

...
