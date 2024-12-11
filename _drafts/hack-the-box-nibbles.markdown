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

### Script scan with default set

Knowing what ports the target is listening on, we can leverage the [Nmap Scripting Engine](https://nmap.org/book/nse.html) (or NSE for short) to perform a script scan using the [default NSE category](https://nmap.org/nsedoc/categories/default.html). The Nmap Scripting Engine is designed to automate various tasks, and the default set allows Nmap to gather even more information about the target such as [Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS).

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

### Script scan with http-enum

Recall that the target host is running an HTTP server. Let's try to enumerate pages and directories exposing interesting data with the [http-enum NSE script](https://nmap.org/nsedoc/scripts/http-enum.html). The http-enum NSE script is not part of the default NSE category, so we must use the `--script=http-enum` option.

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

Our previous script scan with http-enum was unsuccessful, so let's search for interesting files and directories using gobuster instead. [Gobuster](https://github.com/OJ/gobuster) is a tool used to brute-force and find files, directories, subdomains, and other assets. In order to brute-force files and directories, we use the directory/file enumeration mode `dir`.

Enter the following command into the terminal to perform directory/file enumeration:

```
$ gobuster dir -u 10.10.10.75 -w ./common.txt
```

The preceding command consists of the following elements:

- `gobuster` is the command name.
- `dir` is the mode to perform directory/file enumeration.
- `-u` is the flag to specify the target URL.
- `10.10.10.75` is the target URL.
- `-w` is the flag to specify the path to the wordlist.
- `./common.txt` is the path to the [common.txt](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/common.txt) wordlist from SecLists/Discovery/Web-Content.

The output is similar to the following:

```
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.75
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                ./common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 295]
/.htpasswd            (Status: 403) [Size: 295]
/.hta                 (Status: 403) [Size: 290]
/index.html           (Status: 200) [Size: 93]
/server-status        (Status: 403) [Size: 299]

===============================================================
Finished
===============================================================
```

### Manual testing

At first glance, there doesn't appear to be much in the home page of the web application listening on TCP port 80. However, that turns out to not be the case after inspecting its source code. An HTML comment at the bottom of the source code mentions a `/nibbleblog/` directory.

```
<b>Hello world!</b>














<!-- /nibbleblog/ directory. Nothing interesting here! -->

```

Visit the web application at `http://10.10.10.75/nibbleblog` and ...

<!-- TODO (ricardoapl): Explain re-run of gobuster with path to blog -->

Now that we know ...

Enter the following command into the terminal to perform directory/file enumeration:

```
$ gobuster dir -u 10.10.10.75/nibbleblog -w ./common.txt
```

The preceding command consists of the following elements:

- `gobuster` is the command name.
- `dir` is the mode to perform directory/file enumeration.
- `-u` is the flag to specify the target URL.
- `10.10.10.75/nibbleblog` is the target URL.
- `-w` is the flag to specify the path to the wordlist.
- `./common.txt` is the path to the [common.txt](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/common.txt) wordlist from SecLists/Discovery/Web-Content.

The output is similar to the following:

```
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.75/nibbleblog
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                ./common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 306]
/.hta                 (Status: 403) [Size: 301]
/.htpasswd            (Status: 403) [Size: 306]
/README               (Status: 200) [Size: 4628]
/admin                (Status: 301) [Size: 321] [--> http://10.10.10.75/nibbleblog/admin/]
/admin.php            (Status: 200) [Size: 1401]
/content              (Status: 301) [Size: 323] [--> http://10.10.10.75/nibbleblog/content/]
/index.php            (Status: 200) [Size: 2987]
/languages            (Status: 301) [Size: 325] [--> http://10.10.10.75/nibbleblog/languages/]
/plugins              (Status: 301) [Size: 323] [--> http://10.10.10.75/nibbleblog/plugins/]
/themes               (Status: 301) [Size: 322] [--> http://10.10.10.75/nibbleblog/themes/]
Progress: 4735 / 4736 (99.98%)
===============================================================
Finished
===============================================================
```

<!-- XXX (ricardoapl): We'll come back to gobuster once we have access to a (session) cookie (-c), so we can search for protected pages and directories -->

## Vulnerability assessment

<!-- XXX (ricardoapl): Talk about searchsploit and NIST -->

<!-- TODO (ricardoapl): Add reference and/or quote to https://nvd.nist.gov/vuln/detail/CVE-2019-11231 -->

In this stage, ...

## Exploitation

In this stage, ...

### Manual exploitation

<!-- TODO (ricardoapl): Demonstrate manual exploitation -->

Cras eget mauris molestie, pretium urna quis, tristique ipsum. Praesent at lectus pretium, laoreet mi id, porttitor orci. Curabitur luctus eleifend neque, nec faucibus ligula bibendum in. Vivamus sollicitudin feugiat turpis, pulvinar vestibulum tellus lacinia at. Suspendisse potenti. Donec a sollicitudin diam. Suspendisse molestie molestie risus, nec tempor quam. Proin pellentesque molestie congue. In hac habitasse platea dictumst.

### Using the Metasploit Framework

<!-- TODO (ricardoapl): Demonstrate alternative exploitation with metasploit -->

Mauris sollicitudin mi neque, a placerat justo hendrerit nec. Nullam sapien nisi, efficitur at ante vel, blandit consequat ex. Nulla faucibus, ante quis ullamcorper gravida, ipsum mauris tempus nibh, ut efficitur lectus elit a lacus. Cras fermentum quis lorem vitae dapibus. Curabitur varius sodales orci eget dapibus. Nullam fermentum lacus a libero varius, suscipit eleifend libero efficitur. Suspendisse non nulla sapien. Curabitur eu metus nisl. Maecenas sagittis eros felis, tincidunt vestibulum lectus finibus vitae. Praesent mattis quam dui, ut suscipit sapien viverra et. Nulla sit amet fermentum odio. Nulla commodo mi volutpat est iaculis elementum. Suspendisse potenti. Donec vitae sollicitudin eros.

## Post-exploitation

In this stage, ...

## Conclusion

Suspendisse scelerisque non nisl vel lacinia. Curabitur pellentesque, eros id consectetur commodo, sem nisl accumsan augue, a sodales diam est sed tortor. Suspendisse varius non turpis at rutrum. Orci varius natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus. Donec vel suscipit nulla, vel consequat ante. Praesent semper, neque ac tincidunt fringilla, erat lacus dapibus arcu, id vestibulum dolor eros molestie risus. Morbi porttitor nisl non mi sagittis pretium. Nunc ullamcorper faucibus lorem ut volutpat. Sed imperdiet nisl id lorem auctor congue. Etiam dapibus purus blandit elit vestibulum, scelerisque viverra lorem semper. Suspendisse vehicula sem eget facilisis iaculis. Vivamus imperdiet risus ac diam vulputate, ut consequat nunc gravida.
