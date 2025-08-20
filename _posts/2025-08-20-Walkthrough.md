---
title: "THM | Basic Pentration Testing"
date: 2025-08-20 
categories: [Walkthroughs]
tags: [ctf, easy, tryhackme]
---
# Enumeration

### Nmap_scan 
As usual, I start with Nmap’s aggressive scan: `nmap -A <IP>`. The aggressive scan provides a wealth of details with just a single command.The following TCP ports were found to be open. Then there is enumeration of smb.

![Nmap Scan](/assets/ctf/THM/BasicPentset/nmap-scan.png)

### Web_enumeration 
Upon visiting the URL, I found the site to be under maintainance with "Please check back later" instruction! But my hacker sense sensed something fishy. So I inspected the website's source and found a comment saying "Check our dev note section if you need to know what to work on".

![web](/assets/ctf/THM/BasicPentset/web-inspect.png)

So, I ran gobusted scan to find hidden directories. 

    <gobuster dir -u http://<target-ip-address>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -o>

* `dir` - Run this directory to scan.
* `-u http:// target-ip/` - Target URL where we want to find the hidden directory.
* `-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`- The wordlist Gobuster will use to guess directories and files. Each line in the wordlist is tried as a possible directory.
* `-t 64` - Number of concurrent threads to speed up scanning.
* `-o gobuster-bruteforce` - Save the results to a file for later review.

![gobuster](/assets/ctf/THM/BasicPentset/gobuster.png)
Upon visiting the /development directory, we find two initials, "K" and "J," and a weak password hash.
![for-j](/assets/ctf/THM/BasicPentset/for-j.png)
### SMB Enumeration

After running `enum4linux -a`. I found that this machine is running samba where there is share called Anonymous which give us loging without password. 
![anonymous](/assets/ctf/THM/BasicPentset/anonymous.png)
Then i try to access without password using this command.

    <smbclient //target_machine-ip/Anonymous -N>

When I get in I use `ls` command to see what's in, then I find `staff.txt` which i copy in my machine using `get staff.txt`.

![enum](/assets/ctf/THM/BasicPentset/smb.png)

When I see what's on staff.txt there was to username `jan` and `kay`.
![smb](/assets/ctf/THM/BasicPentset/staff.png)

### Brute Force SSH Login
From above information I know that there is two username `jan` and `kay` which help us to brute force using hydra. Where I found the password of `jan`.

![pass](/assets/ctf/THM/BasicPentset/pass.png)

### Linux Enumeration
