---
title: "THM | Basic Penetration Testing"
date: 2025-08-20
categories: [Walkthroughs]
tags: [ctf, easy, tryhackme]
---

# Enumeration

### Nmap_scan
As usual, I started with Nmap’s aggressive scan: `nmap -A <IP>`. The aggressive scan provides a wealth of information with a single command. The following TCP ports were found open, which were later enumerated for SMB.

![Nmap Scan](/assets/ctf/THM/BasicPentset/nmap-scan.png)

### Web_enumeration
Upon visiting the URL, I found the site under maintenance with the message: "Please check back later." However, my hacker instincts sensed something suspicious. Inspecting the website’s source revealed a comment: "Check our dev note section if you need to know what to work on."

![web](/assets/ctf/THM/BasicPentset/web-inspect.png)

I then ran a Gobuster scan to find hidden directories:

    <gobuster dir -u http://<target-ip-address>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -o>

* `dir` - Run a directory scan.  
* `-u http://<target-ip>/` - Target URL to scan for hidden directories.  
* `-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt` - Wordlist used to guess directories and files.  
* `-t 64` - Number of concurrent threads to speed up the scan.  
* `-o gobuster-bruteforce` - Save results to a file for later review.

![gobuster](/assets/ctf/THM/BasicPentset/gobuster.png)

Visiting the `/development` directory, I found two initials, "K" and "J," along with a weak password hash.

![for-j](/assets/ctf/THM/BasicPentset/for-j.png)

### SMB Enumeration
Running `enum4linux -a`, I discovered that the machine runs Samba with a share called `Anonymous` allowing login without a password.

![anonymous](/assets/ctf/THM/BasicPentset/anonymous.png)

I accessed it using:

    <smbclient //target_machine-ip/Anonymous -N>

Once inside, I used `ls` to view the contents and found `staff.txt`, which I copied locally using `get staff.txt`.

![enum](/assets/ctf/THM/BasicPentset/smb.png)

Examining `staff.txt` revealed two usernames: `jan` and `kay`.

![smb](/assets/ctf/THM/BasicPentset/staff.png)

### Brute Force SSH Login
Using the discovered usernames, `jan` and `kay`, I performed a brute-force attack with Hydra and successfully obtained `jan`'s password.

![pass](/assets/ctf/THM/BasicPentset/pass.png)

### Linux Enumeration
After logging in as `jan`, I began enumerating the system to identify potential privilege escalation paths.

![ssh](/assets/ctf/THM/BasicPentset/sshjan.png)

First, I listed home directories to identify other users and found `kay`:

    <ls -la /home/>

![pass](/assets/ctf/THM/BasicPentset/userkay.png)

Inside Kay’s directory, there was a `pass.bank` file for potential password backups, but permission was denied.

![fail](/assets/ctf/THM/BasicPentset/fail.png)

I then checked for world-readable files in Kay’s directory:

    find /home/kay -type f -perm -o+r

![find](/assets/ctf/THM/BasicPentset/find.png)

I copied the private key to a temporary location and adjusted permissions:

    cp /home/kay/.ssh/id_rsa /tmp/kay_id_rsa
    chmod 600 /tmp/kay_id_rsa

An attempt to log in as Kay failed because the key was passphrase-protected:

    ssh -i /tmp/kay_id_rsa kay@<target-ip>

![passphrase](/assets/ctf/THM/BasicPentset/passphrase.png)

### Cracking the SSH Key Passphrase
After locating Kay’s private SSH key, the next step was to crack its passphrase:

**Transfer the key to my local machine**  

    scp jan@10.201.117.94:/tmp/kay_id_rsa ~/Downloads/kay_id_rsa

![scp](/assets/ctf/THM/BasicPentset/scp.png)

**Convert the SSH key for John the Ripper**  
John the Ripper cannot read SSH keys directly, so I converted it using `ssh2john`:

    ssh2john ~/Downloads/kay_id_rsa > kay_id_rsa.hash

**Run John the Ripper**

    sudo john --format=ssh --wordlist=/usr/share/wordlists/rockyou.txt ~/Downloads/kay_id_rsa.hash

![run](/assets/ctf/THM/BasicPentset/run.png)

**Check the cracked passphrase**

    sudo john --show ~/Downloads/kay_id_rsa.hash

![output](/assets/ctf/THM/BasicPentset/output.png)

**Log in as Kay using the SSH key**  
Enter the passphrase `beeswax` when prompted. Once logged in, I found `pass.bak` in Kay’s directory, which revealed the final password.

     ssh -i ~/Downloads/kay_id_rsa kay@target-ip

![kayssh](/assets/ctf/THM/BasicPentset/kayssh.png)

![kayssh2](/assets/ctf/THM/BasicPentset/kayssh2.png)

### Conclusion
In this box, I exploited multiple misconfigurations:

* Nmap and SMB enumeration revealed open services and accessible shares.  
* Gobuster uncovered hidden directories containing sensitive development notes.  
* The anonymous SMB share leaked usernames, which were used for SSH brute-force attacks.  
* Kay’s SSH private key was retrieved and the passphrase cracked to gain user access.  
* Linux enumeration exposed potential privilege escalation paths to root.  

This challenge highlights the importance of securing file shares, using strong and unique passwords, protecting private SSH keys with strong passphrases, and regularly auditing user privileges to prevent unauthorized access.
