---
layout: post
title: Notes on Hack The Box - Nibbles
---

## Summary

In this post, I share some notes from my attempt at solving the [Nibbles](https://app.hackthebox.com/machines/Nibbles) machine from [Hack The Box](https://www.hackthebox.com). The following sections describe my solution to this machine in the context of the penetration testing stages presented in the [Penetration Testing Process](https://academy.hackthebox.com/module/90) module from [Hack The Box Academy](https://academy.hackthebox.com), namely:

- Information gathering
- Vulnerability assessment
- Exploitation
- Post-exploitation

I've omitted the pre-engagement, lateral movement, proof-of-concept, and post-engagement stages because they don't seem to be relevant or applicable.

## Information gathering

In this stage, we gather as much information as possible about our target. Information gathering can be divided into the following categories:

- Open-source intelligence (or OSINT for short)
- Infrastructure enumeration
- Service enumeration
- Host enumeration

There's not much to do when it comes to open-source intelligence or infrastructure enumeration, so I will omit those categories.

We are given the IP address of the target host, and we are told that it's running a Linux distribution as its operating system. In the following sections, I will describe how I've identified:

- Open ports and services
- Service versions
- Operating system

### TCP SYN scan

<!-- TODO (ricardoapl): Explain with https://nmap.org/book/synscan.html (root) -->
<!-- TODO (ricardoapl): Explain with https://nmap.org/book/scan-methods-connect-scan.html (non-root) -->

Enter the following command into the terminal:

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

### TCP SYN scan whole port range

Enter the following command into the terminal:

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

### Script scan with default set

<!-- TODO (ricardoapl): Explain with https://nmap.org/book/nse-usage.html#nse-categories -->

Enter the following command into the terminal:

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

### Script scan with http-enum

<!-- TODO (ricardoapl): Explain with https://nmap.org/nsedoc/scripts/http-enum.html -->

Enter the following command into the terminal:

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

<!-- TODO (ricardoapl): Add gobuster scan -->

## Vulnerability assessment

...

## Exploitation

...

## Post-exploitation

...

## Conclusion

...
